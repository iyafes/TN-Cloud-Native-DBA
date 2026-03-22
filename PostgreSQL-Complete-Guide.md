# PostgreSQL 16 — Complete Study Guide

> **Environment:** Rocky Linux 9 | PostgreSQL 16
> **লক্ষ্য:** Architecture বোঝা থেকে শুরু করে Production HA cluster চালানো পর্যন্ত সব

---

## Table of Contents

1. **[Architecture](#1-architecture)**
   - 1.1 [PostgreSQL কীভাবে কাজ করে — Big Picture](#11-postgresql-কীভাবে-কাজ-করে--big-picture)
   - 1.2 [Process Model — Connection এলে কী হয়](#12-process-model--connection-এলে-কী-হয়)
   - 1.3 [Shared Memory — মস্তিষ্কের মতো](#13-shared-memory--মস্তিষ্কের-মতো)
   - 1.4 [Background Processes — নীরবে কাজ করে](#14-background-processes--নীরবে-কাজ-করে)
   - 1.5 [File System Layout](#15-file-system-layout)
   - 1.6 [WAL — Write-Ahead Log](#16-wal--write-ahead-log)
   - 1.7 [MVCC — একই Table এ Multiple Versions](#17-mvcc--একই-table-এ-multiple-versions)
   - 1.8 [VACUUM — PostgreSQL এর Unique Requirement](#18-vacuum--postgresql-এর-unique-requirement)
   - 1.9 [Complete Read Path (Full Cycle)](#19-complete-read-path-full-cycle)
   - 1.10 [Complete Write Path (Full Cycle)](#110-complete-write-path-full-cycle)
   - 1.11 [UPDATE Operation](#111-update-operation--postgresql-এ-update-মানে-delete--insert)
   - 1.12 [DELETE Operation](#112-delete-operation)
   - 1.13 [ROLLBACK Operation](#113-rollback-operation)
   - 1.14 [Crash Recovery](#114-crash-recovery)
   - 1.15 [Concurrent Read + Write (MVCC in Action)](#115-concurrent-read--write-mvcc-in-action)
   - 1.16 [Component Summary](#116-component-summary)

2. **[Configuration & Parameters](#2-configuration--parameters)**
   - 2.1 [Configuration Files — তিনটা গুরুত্বপূর্ণ File](#21-configuration-files--তিনটা-গুরুত্বপূর্ণ-file)
   - 2.2 [Memory Parameters](#22-memory-parameters)
   - 2.3 [WAL Parameters](#23-wal-parameters)
   - 2.4 [Connection Parameters](#24-connection-parameters)
   - 2.5 [Query Planner Parameters — Partition-wise, Parallel, JIT](#25-query-planner-parameters)
   - 2.6 [Logging Parameters](#26-logging-parameters)
   - 2.7 [Replication Parameters — hot_standby, max_standby_streaming](#27-replication-parameters)
   - 2.8 [ALTER SYSTEM — Persistent Runtime Change](#28-alter-system--persistent-runtime-change)
   - 2.9 [Production postgresql.conf Template](#29-production-postgresqlconf-16gb-ram-ssd)
   - 2.10 [OS Prerequisites — Install এর আগে](#210-os-prerequisites--postgresql-install-এর-আগে)
   - 2.11 [pg_hba.conf — Authentication Deep Dive](#211-pg_hbaconf--authentication-deep-dive)
   - 2.12 [Configuration Tuning Guide — RAM অনুযায়ী](#212-configuration-tuning-guide--ram-অনুযায়ী)
   - 2.13 [Parameter Tuning Workflow](#213-parameter-tuning-workflow)

3. **[Data Types](#3-data-types)**
   - 3.1 [Numeric Types](#31-numeric-types)
   - 3.2 [Character Types](#32-character-types)
   - 3.3 [Date and Time Types](#33-date-and-time-types)
   - 3.4 [Boolean](#34-boolean)
   - 3.5 [JSON vs JSONB](#35-json-vs-jsonb--কোনটা-কখন)
   - 3.6 [Arrays](#36-arrays--column-এ-list-store)
   - 3.7 [UUID](#37-uuid)
   - 3.8 [Range Types](#38-range-types--period-এবং-range-data)
   - 3.9 [Practical Table Design](#39-practical-table-design)

4. **[Indexes](#4-indexes)**
   - 4.1 [Index কেন দরকার](#41-index-কেন-দরকার)
   - 4.2 [B-Tree Index — Default](#42-b-tree-index--default-এবং-সবচেয়ে-common)
   - 4.3 [Hash Index — Equality Only](#43-hash-index--equality-only-faster)
   - 4.4 [GIN Index — Full-Text, Array, JSONB](#44-gin-index--full-text-array-jsonb)
   - 4.5 [GiST Index — Geometric, Range](#45-gist-index--geometric-range-full-text)
   - 4.6 [BRIN Index — Large Sequential Table](#46-brin-index--large-sequential-table-এ)
   - 4.7 [Partial Index — Subset of Rows](#47-partial-index--subset-of-rows)
   - 4.8 [Expression Index — Function এর Result এ](#48-expression-index--function-এর-result-এ-index)
   - 4.9 [Covering Index — Index Only Scan](#49-covering-index--index-only-scan)
   - 4.10 [EXPLAIN — Execution Plan Analysis](#410-explain--execution-plan-analysis)
   - 4.11 [Index Design Rules](#411-index-design-rules)

5. **[Transactions & Locking](#5-transactions--locking)**
   - 5.1 [Transaction কী এবং কেন](#51-transaction-কী-এবং-কেন)
   - 5.2 [ACID Properties](#52-acid-properties)
   - 5.3 [Basic Commands](#53-basic-commands)
   - 5.4 [Isolation Levels](#54-isolation-levels)
   - 5.5 [MVCC Deep Dive](#55-mvcc-deep-dive)
   - 5.6 [Lock Types](#56-lock-types)
   - 5.7 [Deadlock](#57-deadlock)
   - 5.8 [Advisory Locks](#58-advisory-locks--application-level-custom-locks)
   - 5.9 [Lock Monitoring](#59-lock-monitoring)

6. **[Query Optimization & Performance](#6-query-optimization)**
   - 6.1 [Statistics and ANALYZE](#61-statistics-and-analyze)
   - 6.2 [pg_stat_statements](#62-pg_stat_statements--built-in-query-tracking)
   - 6.3 [pg_stat_monitor (Percona)](#63-pg_stat_monitor--enhanced-query-analytics-percona)
   - 6.4 [EXPLAIN — Full Usage](#64-explain--full-usage)
   - 6.5 [Common Slow Patterns এবং Fix](#65-common-slow-patterns-এবং-fix)
   - 6.6 [Planner Settings Override](#66-planner-settings-override)
   - 6.7 [Table Partitioning](#67-table-partitioning)
   - 6.8 [CTEs (Common Table Expressions)](#68-ctes-common-table-expressions)
   - 6.9 [Parallel Query](#69-parallel-query)
   - 6.10 [Optimization Checklist](#610-optimization-checklist)
   - 6.11 [JOIN Optimization — Deep Dive](#611-join-optimization--deep-dive)
   - 6.12 [Aggregate এবং GROUP BY Optimization](#612-aggregate-এবং-group-by-optimization)
   - 6.13 [Write Performance (INSERT/UPDATE/DELETE)](#613-write-performance-insertupdatedelete)
   - 6.14 [Connection Pool Performance](#614-connection-pool-performance)
   - 6.15 [Autovacuum Performance Tuning](#615-autovacuum-performance-tuning)
   - 6.16 [VACUUM এবং Table Bloat Management](#616-vacuum-এবং-table-bloat-management)
   - 6.17 [Hardware-Level Tuning](#617-hardware-level-tuning)
   - 6.18 [pgBench — Load Testing](#618-pgbench--load-testing)
   - 6.19 [Query Rewrite Techniques](#619-query-rewrite-techniques)
   - 6.20 [Performance Troubleshooting Workflow](#620-performance-troubleshooting-workflow)

7. **[Backup & Recovery](#7-backup--recovery)**
   - 7.0 [Backup Types — Logical vs Physical, Full/Diff/Incr](#70-backup-types--সব-কিছুর-ভিত্তি)
   - 7.1 [pg_dump — Single Database Logical Backup](#71-pg_dump--single-database-logical-backup)
   - 7.2 [pg_dumpall — Cluster-wide Backup](#72-pg_dumpall--cluster-wide-backup)
   - 7.3 [pg_restore — Restore Backup](#73-pg_restore--restore-backup)
   - 7.4 [pg_basebackup — Physical Backup](#74-pg_basebackup--physical-backup)
   - 7.5 [WAL Archiving](#75-wal-archiving)
   - 7.6 [Point-in-Time Recovery (PITR) — Time, LSN, XID, Named Restore Point](#76-point-in-time-recovery-pitr)
   - 7.7 [pgBackRest — Production Backup Solution](#77-pgbackrest--production-backup-solution)
   - 7.8 [Barman — Multi-Server Backup](#78-barman--multi-server-backup-management)
   - 7.9 [Backup Best Practices](#79-backup-best-practices)
   - 7.10 [Backup Verification](#710-backup-verification--নিশ্চিত-করো-backup-কাজ-করবে)
   - 7.11 [Restore Testing — Monthly Drill](#711-restore-testing--backup-এর-সবচেয়ে-গুরুত্বপূর্ণ-part)
   - 7.12 [Backup Encryption](#712-backup-encryption)
   - 7.13 [Backup Monitoring এবং Alerting](#713-backup-monitoring-এবং-alerting)
   - 7.14 [Backup Tool Comparison](#714-backup-tool-comparison--কোনটা-কখন)

8. **[Monitoring](#8-monitoring)**
   - 8.1 [OS Level + pg_prewarm — Buffer Pool Warm after Restart](#81-os-level)
   - 8.2 [pg_stat_activity — Connection Monitoring](#82-pg_stat_activity--connection-এবং-query-monitoring)
   - 8.3 [pg_stat_user_tables — Table Health](#83-pg_stat_user_tables--table-health)
   - 8.4 [Replication Monitoring](#84-replication-monitoring)
   - 8.5 [VACUUM Monitoring](#85-vacuum-monitoring)
   - 8.6 [Lock Monitoring](#86-lock-monitoring)
   - 8.7 [Index Usage Monitoring](#87-index-usage-monitoring)
   - 8.7b [amcheck — Index Corruption Detection](#87b-amcheck--index-corruption-detection)
   - 8.7c [Operation Progress Monitoring](#87c-operation-progress-monitoring)
   - 8.8 [Percona PMM](#88-percona-monitoring-and-management-pmm)
   - 8.9 [Monitoring Dashboard Script](#89-monitoring-dashboard-script)
   - 8.10 [Key Metrics Thresholds](#810-key-metrics-thresholds)
   - 8.11 [pg_stat_database — Database-Level Statistics](#811-pg_stat_database--database-level-statistics)
   - 8.12 [pg_stat_bgwriter — Background Writer Monitoring](#812-pg_stat_bgwriter--background-writer-monitoring)
   - 8.12b [pg_stat_io — I/O Statistics (PG 16)](#812b-pg_stat_io--postgresql-16-এর-io-statistics)
   - 8.13 [Log Analysis](#813-log-analysis)
   - 8.14 [Alerting Setup — Custom Alerts](#814-alerting-setup--custom-alerts)

9. **[Security](#9-security)**
   - 9.1 [Authentication — pg_hba.conf](#91-authentication--pg_hbaconf)
   - 9.2 [Roles & Privileges](#92-roles--privileges)
   - 9.3 [Row Level Security (RLS)](#93-row-level-security-rls)
   - 9.4 [Column Level Security](#94-column-level-security)
   - 9.5 [SSL/TLS](#95-ssltls)
   - 9.6 [pgcrypto — Database Encryption](#96-pgcrypto--database-level-encryption)
   - 9.7 [pgAudit — Audit Logging](#97-pgaudit--audit-logging)
   - 9.8 [Security Checklist](#98-security-checklist)

10. **[High Availability & Replication](#10-high-availability--replication)**
    - 10.1 [RPO এবং RTO](#101-rpo-এবং-rto--ha-design-এর-ভিত্তি)
    - 10.2 [Streaming Replication — কীভাবে কাজ করে](#102-streaming-replication--কীভাবে-কাজ-করে)
    - 10.3 [Install থেকে Streaming Replication — End-to-End](#103-install-থেকে-streaming-replication--end-to-end)
    - 10.4 [Replication Slots](#104-replication-slots)
    - 10.5 [Synchronous Replication](#105-synchronous-replication)
    - 10.6 [Logical Replication](#106-logical-replication)
    - 10.7 [Patroni + etcd + HAProxy — Production HA](#107-patroni--etcd--haproxy--production-ha)
    - 10.8 [pgBouncer — Connection Pooler](#108-pgbouncer--connection-pooler)
    - 10.9 [pgPool-II](#109-pgpool-ii)
    - 10.10 [Failover](#1010-failover)
    - 10.11 [Replication Topology Comparison](#1011-replication-topology-comparison)
    - 10.12 [Troubleshooting Replication](#1012-troubleshooting-replication)
    - 10.13 [Quick Reference Commands](#1013-quick-reference-commands)
    - 10.14 [Delayed Standby — Accidental Data Loss Protection](#1014-delayed-standby--accidental-data-loss-protection)
    - 10.15 [Cascading Replication](#1015-cascading-replication)
    - 10.16 [Read Scaling Architecture](#1016-read-scaling-architecture)
    - 10.17 [HA Troubleshooting Scenarios](#1017-ha-troubleshooting-scenarios)

11. **[Percona for PostgreSQL](#11-percona-for-postgresql)**
    - 11.1 [Percona Distribution — Overview](#111-percona-distribution-for-postgresql--overview)
    - 11.2 [pg_stat_monitor — Deep Dive](#112-pg_stat_monitor--deep-dive)
    - 11.3 [PMM Overview](#113-percona-monitoring-and-management-pmm)
    - 11.4 [PMM Setup — End-to-End](#114-pmm-setup--end-to-end)
    - 11.5 [PMM Query Analytics (QAN)](#115-pmm-query-analytics-qan--ব্যবহার-করার-guide)
    - 11.6 [pg_stat_monitor vs pg_stat_statements](#116-pg_stat_monitor-vs-pg_stat_statements)
    - 11.7 [Percona Toolkit for PostgreSQL](#117-percona-toolkit-for-postgresql)
    - 11.8 [Percona vs Vanilla PostgreSQL](#118-percona-vs-vanilla-postgresql)

12. **[PL/pgSQL — Functions, Procedures & Triggers](#12-plpgsql--functions-procedures--triggers)**
    - 12.1 [PL/pgSQL কী এবং কেন](#121-plpgsql-কী-এবং-কেন)
    - 12.2 [Function — Basic Structure](#122-function--basic-structure)
    - 12.3 [Control Structures — IF, LOOP, CASE](#123-control-structures--if-loop-case)
    - 12.4 [Exception Handling](#124-exception-handling)
    - 12.5 [Stored Procedures (PG 11+)](#125-stored-procedures-postgresql-11)
    - 12.6 [Return Types — TABLE, SETOF, OUT parameters](#126-return-types--table-setof-record)
    - 12.7 [Triggers — Auto-update, Audit, Validation](#127-triggers--data-change-এ-automatic-action)
    - 12.8 [Function Performance এবং Volatility](#128-function-performance-এবং-volatility)
    - 12.9 [Useful Built-in Functions — Quick Reference](#129-useful-built-in-functions--quick-reference)

13. **[Major Version Upgrade](#13-major-version-upgrade)**
    - 13.1 [Upgrade Methods — Overview](#131-upgrade-methods--overview)
    - 13.2 [Method 1 — pg_dump + pg_restore](#132-method-1--pg_dump--pg_restore)
    - 13.3 [Method 2 — pg_upgrade (Large DB)](#133-method-2--pg_upgrade-recommended-for-large-db)
    - 13.4 [Method 3 — Logical Replication (Zero-Downtime)](#134-method-3--logical-replication-zero-downtime)
    - 13.5 [Pre-upgrade Checklist এবং Common Issues](#135-pre-upgrade-checklist-এবং-common-issues)

14. **[Extensions — PostgreSQL Ecosystem](#14-extensions--postgresql-ecosystem)**
    - 14.1 [Extension কী এবং Management](#141-extension-কী-এবং-কীভাবে-কাজ-করে)
    - 14.2 [pg_cron — Scheduled Jobs](#142-pg_cron--database-থেকে-scheduled-jobs)
    - 14.3 [pg_partman — Automatic Partitioning](#143-pg_partman--automatic-partition-management)
    - 14.4 [hypopg — Hypothetical Indexes](#144-hypopg--hypothetical-indexes)
    - 14.5 [auto_explain — Slow Query Plan Capture](#145-auto_explain--slow-query-এর-plan-auto-capture)
    - 14.6 [pg_trgm — Fuzzy Text Search](#146-pg_trgm--trigram-similarity-search)
    - 14.7 [PostGIS — Geographic Data](#147-postgis--geographic-data)
    - 14.8 [TimescaleDB — Time-Series Data](#148-timescaledb--time-series-data)
    - 14.9 [Extensions Setup Script](#149-pg_stat_statements-এবং-extensions-setup-script)
    - 14.10 [Extension Quick Reference](#1410-extension-quick-reference)

15. **[Troubleshooting Scenarios এবং Schema Management](#15-troubleshooting-scenarios-এবং-schema-management)**
    - 15.1 ["Database Suddenly Slow" — Systematic Diagnosis](#151-database-suddenly-slow--systematic-diagnosis)
    - 15.2 [OOM Killer PostgreSQL Kill করলে](#152-oom-killer-postgresql-kill-করলে)
    - 15.3 [Data Corruption Detection এবং Recovery](#153-data-corruption-detection-এবং-recovery)
    - 15.4 [Connection Exhaustion](#154-connection-exhaustion-too-many-connections)
    - 15.5 [Schema Management Best Practices](#155-schema-management-best-practices)
    - 15.6 [MySQL থেকে PostgreSQL Migration Gotchas](#156-mysql-থেকে-postgresql-migration-gotchas)
    - 15.7 [Disk Full — Emergency Recovery](#157-disk-full--emergency-recovery)
    - 15.8 [PostgreSQL Start হচ্ছে না — Diagnosis](#158-postgresql-start-হচ্ছে-না--diagnosis)
    - 15.9 [Autovacuum এবং XID Wraparound Emergency](#159-autovacuum-এবং-xid-wraparound-emergency)
    - 15.10 [Production DBA Daily Checklist Script](#1510-production-dba-daily-checklist)

16. **[Ultimate Quick Reference](#16-ultimate-quick-reference)**
    - 16.1 [Installation — Rocky Linux 9](#161-installation--rocky-linux-9)
    - 16.2 [psql — সব দরকারি Commands](#162-psql--সব-দরকারি-commands)
    - 16.3 [Database এবং User Management](#163-database-এবং-user-management)
    - 16.4 [Table Operations](#164-table-operations)
    - 16.5 [Query Patterns — Most Used](#165-query-patterns--most-used)
    - 16.6 [Performance Diagnosis — One-liners](#166-performance-diagnosis--one-liners)
    - 16.7 [Maintenance Commands](#167-maintenance-commands)
    - 16.8 [Config Quick Changes](#168-config-quick-changes)
    - 16.9 [Backup Quick Reference](#169-backup-quick-reference)
    - 16.10 [Replication Quick Reference](#1610-replication-quick-reference)
    - 16.11 [Common Error Messages](#1611-common-error-messages--কী-মানে-কী-করবে)

17. **[Lock Contention — Deep Dive](#17-lock-contention--deep-dive)**
    - 17.1 [Lock কেন Production এর সবচেয়ে বড় সমস্যা](#171-lock-কেন-production-এর-সবচেয়ে-বড়-সমস্যা)
    - 17.2 [PostgreSQL Lock Hierarchy — সম্পূর্ণ চিত্র](#172-postgresql-lock-hierarchy--সম্পূর্ণ-চিত্র)
    - 17.3 [কোন DDL কোন Lock নেয়](#173-কোন-ddl-কোন-lock-নেয়)
    - 17.4 [Zero-Downtime DDL — Production Safe Patterns](#174-zero-downtime-ddl--production-safe-patterns)
    - 17.5 [Lock Monitoring এবং Kill](#175-lock-monitoring-এবং-kill)
    - 17.6 [Lock Timeout এবং Prevention](#176-lock-timeout-এবং-prevention)
    - 17.7 [Production ALTER TABLE Checklist](#177-production-alter-table-checklist)

18. **[Sequence Management](#18-sequence-management)**
    - 18.1 [Sequence কী এবং কীভাবে কাজ করে](#181-sequence-কী-এবং-কীভাবে-কাজ-করে)
    - 18.2 [Sequence Gap কেন হয়](#182-sequence-gap-কেন-হয়-এবং-কেন-এটা-normal)
    - 18.3 [Sequence Exhaustion — Critical Problem](#183-sequence-exhaustion--critical-problem)
    - 18.4 [Migration পরে Sequence Reset](#184-migration-পরে-sequence-reset)
    - 18.5 [Sequence Best Practices](#185-sequence-best-practices)

19. **[JSONB Advanced Patterns](#19-jsonb-advanced-patterns)**
    - 19.1 [JSON Path — PG 12+](#191-json-path--pg-12)
    - 19.2 [JSONB Schema Validation এবং Transformation](#192-jsonb-schema-validation-এবং-transformation)

20. **[Full-Text Search — Deep Dive](#20-full-text-search--deep-dive)**
    - 20.1 [tsvector এবং tsquery](#201-tsvector-এবং-tsquery--কীভাবে-কাজ-করে)
    - 20.2 [Full-Text Search Implementation](#202-full-text-search-implementation)
    - 20.3 [Multilingual Full-Text Search](#203-multilingual-full-text-search)
    - 20.4 [pg_trgm দিয়ে Fuzzy Search](#204-pg_trgm-দিয়ে-fuzzy-search)

21. **[Foreign Data Wrappers (FDW)](#21-foreign-data-wrappers-fdw)**
    - 21.1 [postgres_fdw — PostgreSQL থেকে PostgreSQL](#211-postgres_fdw--postgresql-থেকে-postgresql-query)
    - 21.2 [file_fdw — CSV/File থেকে Read](#212-file_fdw--csvfile-থেকে-read)
    - 21.3 [mysql_fdw — MySQL থেকে Read](#213-mysql_fdw--mysql-থেকে-read)

22. **[Extended Statistics এবং Advanced Tuning](#22-extended-statistics-এবং-advanced-planner-tuning)**
    - 22.1 [CREATE STATISTICS — Multi-Column Correlation](#221-create-statistics--multi-column-correlation)
    - 22.2 [Connection Management — Complete Picture](#222-connection-management--complete-picture)
    - 22.3 [Post-Upgrade Checklist](#223-post-upgrade-checklist)

23. **[Advanced PL/pgSQL](#23-advanced-plpgsql)**
    - 23.1 [Dynamic SQL](#231-dynamic-sql)
    - 23.2 [Cursor — Large Result Set Processing](#232-cursor--large-result-set-processing)
    - 23.3 [LISTEN / NOTIFY — Async Messaging](#233-listen--notify--async-messaging)
    - 23.4 [GET DIAGNOSTICS — Operation Result Info](#234-get-diagnostics--operation-result-info)
    - 23.5 [Row-Level Security এ Function](#235-row-level-security-এ-function)

---

# 1. Architecture

## 1.1 PostgreSQL কীভাবে কাজ করে — Big Picture

PostgreSQL কে বুঝতে হলে প্রথমে জানতে হবে এটা MySQL থেকে কোথায় আলাদা। সবচেয়ে বড় পার্থক্য দুটো:

**১. Process-based** — MySQL প্রতিটা connection এ একটা thread তৈরি করে। PostgreSQL প্রতিটা connection এ একটা আলাদা OS **process** তৈরি করে (fork)। এই কারণে PostgreSQL এ connection overhead বেশি — তাই production এ pgBouncer connection pooler ব্যবহার করতে হয়।

**২. MVCC implementation আলাদা** — MySQL পুরনো row version Undo Log এ রাখে। PostgreSQL পুরনো এবং নতুন দুটো version একই table এ রাখে (dead tuple)। এই কারণে PostgreSQL এ **VACUUM** লাগে — MySQL তে লাগে না।

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CLIENT LAYER                                │
│           Application / psql / JDBC / libpq / ORM                  │
└─────────────────────────────┬───────────────────────────────────────┘
                              │ TCP/IP (port 5432) বা Unix Socket
┌─────────────────────────────▼───────────────────────────────────────┐
│                    POSTMASTER (Main Process)                        │
│          সব কিছু manage করে — নতুন connection এলে নতুন             │
│          Backend Process fork করে দেয়                               │
└─────────────────────────────┬───────────────────────────────────────┘
                              │ fork()
┌─────────────────────────────▼───────────────────────────────────────┐
│              BACKEND PROCESS (প্রতিটা connection এর জন্য আলাদা)    │
│       Parser → Rewriter → Planner/Optimizer → Executor              │
└─────────────────────────────┬───────────────────────────────────────┘
                              │ সব backend process একই shared memory ব্যবহার করে
┌─────────────────────────────▼───────────────────────────────────────┐
│                         SHARED MEMORY                               │
│   ┌──────────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│   │  Shared Buffers  │  │ WAL Buffers  │  │  Lock Table / CLOG   │  │
│   │  (Data Cache)    │  │ (Log Cache)  │  │  (Transaction Info)  │  │
│   └──────────────────┘  └──────────────┘  └──────────────────────┘  │
└─────────────────────────────┬───────────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────────┐
│                    BACKGROUND PROCESSES                             │
│  WAL Writer · Checkpointer · Background Writer · Autovacuum         │
│  Stats Collector · WAL Sender · WAL Receiver · Archiver             │
└─────────────────────────────┬───────────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────────┐
│                          DISK / FILE SYSTEM                         │
│   base/ (table data) · pg_wal/ (WAL) · pg_xact/ (transaction log)  │
│   postgresql.conf · pg_hba.conf · pg_control                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 1.2 Process Model — Connection এলে কী হয়

```
Client connect করলে:
─────────────────────────────────────────────
  1. TCP handshake → Postmaster এ পৌঁছায়
  2. Postmaster → fork() → নতুন Backend Process তৈরি
  3. Backend Process:
       → pg_hba.conf এ authentication rule check করে
       → Password verify করে
       → Search path, timezone, settings setup করে
       → Query এর জন্য ready

Client disconnect করলে:
─────────────────────────────────────────────
  Backend Process terminate হয়
  Memory free হয়
  (MySQL এর মতো Thread Cache নেই — তাই pgBouncer দরকার)
```

```sql
-- কতটা process চলছে দেখো
SELECT pid, usename, application_name, client_addr,
       state, backend_type
FROM pg_stat_activity
ORDER BY backend_type, state;

-- Background processes আলাদা করে দেখো
SELECT pid, backend_type, state
FROM pg_stat_activity
WHERE backend_type != 'client backend';
```

**MySQL vs PostgreSQL:**
| | MySQL | PostgreSQL |
|---|---|---|
| Connection model | Thread-based | Process-based (fork) |
| Memory per connection | ~1MB | ~5-10MB |
| Connection pooling | Built-in (Thread Cache) | External (pgBouncer) needed |
| Max practical connections | 1000+ | 200-300 (তারপর pgBouncer) |

---

## 1.3 Shared Memory — মস্তিষ্কের মতো

Shared Memory হলো সব Backend Process এর shared workspace। এখানে যা থাকে সেটা disk থেকে অনেক দ্রুত access করা যায়।

### Shared Buffers — সবচেয়ে গুরুত্বপূর্ণ

Disk থেকে read করা data pages এখানে cache হয়। MySQL এর Buffer Pool এর মতো — কিন্তু পার্থক্য আছে।

```
MySQL Buffer Pool:  RAM এর 70-80% দাও
PostgreSQL Shared Buffers: RAM এর 25% দাও

কেন কম?
PostgreSQL এ OS Page Cache ও কাজ করে।
Disk read হলে OS নিজেও cache করে।
তাই 25% Shared Buffers + OS Cache মিলে effective।
MySQL O_DIRECT দিয়ে OS cache bypass করে, তাই বেশি দিতে হয়।
```

**Clock-Sweep Algorithm** (LRU এর variant):
```
প্রতিটা page এর usage_count আছে (0 থেকে 5)
Page access হলে → usage_count বাড়ে
Evict করতে হলে → usage_count=0 এর page বের করো
usage_count>0 হলে → count কমাও, পরের বার দেখো

কেন LRU নয়?
LRU তে প্রতিটা access এ list reorder লাগে → expensive
Clock-sweep শুধু count update → cheap
```

```sql
-- Buffer hit rate দেখো (99%+ হওয়া উচিত)
SELECT
    ROUND(
        sum(heap_blks_hit) * 100.0 /
        NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0),
    2) AS hit_rate_pct
FROM pg_statio_user_tables;
```

### WAL Buffers

WAL (Write-Ahead Log) এর in-memory buffer। Transaction এর changes প্রথমে এখানে লেখা হয়, তারপর disk এ।

### CLOG (pg_xact) — Transaction Status Registry

প্রতিটা transaction committed নাকি aborted সেটার record।

```
XID 100: COMMITTED
XID 101: ABORTED
XID 102: IN_PROGRESS

MVCC row visibility check এর সময় CLOG দেখতে হয়।
"এই row টা কোন transaction insert করেছে? সে committed?"
```

### Lock Table

কোন process কোন resource lock করেছে সেটার shared registry। সব Backend Process এটা দেখে নতুন lock নেওয়ার আগে।

---

## 1.4 Background Processes — নীরবে কাজ করে

```
Postmaster (Parent Process)
    │
    ├── WAL Writer
    │     কাজ: WAL Buffers → disk এ flush করে
    │     কখন: প্রতি wal_writer_delay (200ms), বা buffer 1/3 full হলে
    │     COMMIT এ synchronously flush করে (synchronous_commit=on)
    │
    ├── Checkpointer
    │     কাজ: Dirty pages (modified but not written) disk এ flush করে
    │     কখন: checkpoint_timeout (5min) বা max_wal_size (1GB) পূর্ণ হলে
    │     Checkpoint LSN record করে → crash recovery তে কাজ লাগে
    │
    ├── Background Writer (bgwriter)
    │     কাজ: Checkpointer এর আগেই কিছু dirty pages flush করে
    │     উদ্দেশ্য: Checkpoint spike কমানো, I/O smooth করা
    │     কখন: প্রতি bgwriter_delay (200ms)
    │
    ├── Autovacuum Launcher
    │     কাজ: কোন table এ VACUUM/ANALYZE দরকার দেখে Worker spawn করে
    │     কখন: প্রতি autovacuum_naptime (1min)
    │     ├── Autovacuum Worker 1 → orders table VACUUM
    │     └── Autovacuum Worker 2 → users table ANALYZE
    │
    ├── Stats Collector
    │     কাজ: Table access, query count, tuple stats collect করে
    │     pg_stat_* views এর data এখান থেকে আসে
    │
    ├── WAL Sender (replication এ)
    │     কাজ: Replica কে WAL stream করে পাঠায়
    │     প্রতিটা Replica এর জন্য একটা WAL Sender
    │
    ├── WAL Receiver (Replica তে)
    │     কাজ: Primary থেকে WAL stream receive করে
    │
    └── Archiver (archive_mode=on হলে)
          কাজ: Completed WAL segment গুলো archive location এ copy করে
          PITR (Point-in-Time Recovery) এর জন্য দরকার
```

---

## 1.5 File System Layout

`$PGDATA` হলো PostgreSQL এর data directory। সব কিছু এখানে।

```
$PGDATA (/var/lib/pgsql/16/data/)
│
├── postgresql.conf       ← Main configuration (সব parameter এখানে)
├── postgresql.auto.conf  ← ALTER SYSTEM এর changes (override করে .conf কে)
├── pg_hba.conf           ← Authentication rules (কে কোথা থেকে connect করতে পারবে)
├── pg_ident.conf         ← OS username → PG username mapping
├── PG_VERSION            ← "16" — PostgreSQL version
├── postmaster.pid        ← Running server এর PID (এই file থাকলে server চলছে)
│
├── base/                 ← সব database এর data files
│   ├── 1/                ← template1 database (OID=1)
│   ├── 16384/            ← mydb database (OID assigned when created)
│   │   ├── 16385         ← users table এর data file (OID = RelFileNode)
│   │   ├── 16385_fsm     ← Free Space Map (INSERT এর জন্য কোথায় space আছে)
│   │   ├── 16385_vm      ← Visibility Map (VACUUM optimization)
│   │   └── 16386         ← users_pkey index এর data file
│   └── ...
│
├── pg_wal/               ← Write-Ahead Log segments (crash recovery + replication)
│   ├── 000000010000000000000001   ← প্রতিটা file 16MB
│   └── 000000010000000000000002
│
├── pg_xact/              ← Transaction commit status (CLOG)
│                           কোন XID committed/aborted/in-progress
│
├── global/               ← Cluster-wide shared tables
│   └── pg_control        ← Critical! Checkpoint info, cluster state
│                           এই file corrupt হলে DB start হবে না
│
└── log/                  ← Server log files (logging_collector=on হলে)
```

**গুরুত্বপূর্ণ files:**
| File/Dir | কাজ | নষ্ট হলে |
|---|---|---|
| `postgresql.conf` | Configuration | Server start হবে না |
| `pg_hba.conf` | Authentication | কেউ connect করতে পারবে না |
| `pg_control` | Cluster state, checkpoint | Recovery সম্ভব নাও হতে পারে |
| `pg_wal/` | WAL segments | Crash recovery কাজ করবে না |
| `pg_xact/` | Transaction status | MVCC ভেঙে পড়বে |

---

## 1.6 WAL — Write-Ahead Log

WAL হলো PostgreSQL এর সবচেয়ে গুরুত্বপূর্ণ mechanism। তিনটা কাজ করে:

```
১. Crash Recovery (Durability):
   Data page disk এ লেখার আগে WAL এ লিখতে হবে।
   Crash হলে WAL replay করে data recover হয়।

২. Streaming Replication:
   WAL segment Replica কে পাঠানো হয়।
   Replica সেই WAL apply করে Primary এর copy হয়।

৩. PITR (Point-in-Time Recovery):
   Base backup + WAL archive = যেকোনো সময়ে restore করা যাবে।
```

### WAL Segment

```
প্রতিটা WAL file = 16MB
Filename: 000000010000000000000001
          │────────│─────────────│
          timeline  log segment number (hex)

Timeline: Failover হলে বাড়ে (1 → 2 → 3)
          কোন timeline এ আছি সেটা বোঝা যায়
```

### LSN — Log Sequence Number

WAL এর position। বইয়ের page number এর মতো।

```sql
-- Primary এর current WAL position
SELECT pg_current_wal_lsn();
-- Result: 0/3A1F2B8 (hex format)

-- কোন WAL file এ আছি
SELECT pg_walfile_name(pg_current_wal_lsn());
-- Result: 000000010000000000000003

-- Replica থেকে কতটা পিছিয়ে আছে (Primary তে চালাও)
SELECT
    application_name,
    pg_size_pretty(sent_lsn - replay_lsn) AS lag_size,
    replay_lag AS lag_time
FROM pg_stat_replication;
```

### `synchronous_commit` — Durability vs Speed Tradeoff

| Value | মানে | Speed | Safety |
|---|---|---|---|
| `on` (default) | COMMIT এর আগে WAL disk এ flush | Slow | সম্পূর্ণ safe |
| `off` | WAL flush এর আগেই COMMIT ack | Fast | শেষ কিছু transaction হারাতে পারে |
| `local` | Local WAL flush, Replica না হলেও ack | Medium | Local crash safe |
| `remote_write` | Replica memory তে লেখা হলে ack | Fast | OS crash এ হারাতে পারে |
| `remote_apply` | Replica apply করলে ack | Slowest | সবচেয়ে safe for replication |

```ini
# Production (banking, e-commerce)
synchronous_commit = on

# High-throughput logging (কিছু data loss acceptable)
synchronous_commit = off
```

---

## 1.7 MVCC — একই Table এ Multiple Versions

**MVCC (Multi-Version Concurrency Control)** মানে একই row এর একাধিক version রাখা — যাতে read এবং write একে অপরকে block না করে।

### MySQL vs PostgreSQL MVCC

```
MySQL MVCC:
  Current version → heap এ (table এ)
  Old versions   → Undo Log এ (আলাদা file)
  Rollback দরকার হলে → Undo Log থেকে পুরনো version আনো
  পুরানো version → InnoDB Purge Thread automatically clean করে
  → VACUUM দরকার নেই

PostgreSQL MVCC:
  সব versions (old + new) → একই heap এ (table এ)
  Old version = "dead tuple" (xmax set হয়ে থাকে)
  Rollback → xmax clear করে (পুরনো tuple আবার visible)
  Dead tuples → VACUUM manually বা autovacuum clean করে
  → VACUUM না চললে table bloat হয়!
```

### Hidden System Columns

প্রতিটা row এ কিছু hidden column থাকে যেগুলো MVCC এর জন্য দরকার:

```sql
-- Hidden columns দেখো
SELECT xmin, xmax, ctid, id, name FROM users LIMIT 5;
```

| Column | মানে |
|---|---|
| `xmin` | যে Transaction ID (XID) এই row INSERT করেছে |
| `xmax` | যে XID এই row DELETE/UPDATE করেছে (0 = কেউ না) |
| `ctid` | Row এর physical location: `(page_number, slot_number)` |

### কীভাবে Visibility Decide হয়

```
Row: xmin=100, xmax=0, name='Alice'

Transaction A (XID=150) SELECT করলে:
  xmin=100 → XID 100 committed? → CLOG check → YES → visible
  xmax=0   → কেউ delete করেনি → still alive
  → 'Alice' দেখাবে ✅

এখন Transaction B (XID=200): UPDATE SET name='Bob'
  Old row: xmin=100, xmax=200  (B delete করেছে)
  New row: xmin=200, xmax=0    (B insert করেছে)

Transaction A এখন আবার SELECT করলে (REPEATABLE READ):
  Old row: xmin=100 (visible), xmax=200 (B committed, >A's snapshot) → visible!
  New row: xmin=200 > A's snapshot (150) → NOT visible
  → 'Alice' দেখাবে (B এর change দেখবে না) ✅

Transaction C (XID=250, new) SELECT করলে:
  Old row: xmax=200, B committed → dead
  New row: xmin=200, committed → visible
  → 'Bob' দেখাবে ✅
```

---

## 1.8 VACUUM — PostgreSQL এর Unique Requirement

VACUUM দরকার কারণ dead tuples জমে table bloat হয়।

### কী কী করে VACUUM

```
VACUUM:
  1. Dead tuples (xmax != 0, committed) scan করে
  2. সেই space free করে (FSM update)
  3. Dead index entries সরায়
  4. Visibility Map update করে (Index Only Scan এর জন্য দরকার)
  5. Transaction ID Wraparound থেকে রক্ষা করে

VACUUM ANALYZE:
  VACUUM + Statistics update (query planner এর জন্য)

VACUUM FULL:
  পুরো table rewrite করে
  Disk space OS কে ফেরত দেয়
  ⚠️ ACCESS EXCLUSIVE lock নেয় — পুরো table block!
  Production এ pg_repack ব্যবহার করো বরং
```

### Transaction ID Wraparound — Critical Problem

```
XID 32-bit = maximum ~2 billion transactions
XID শেষ হয়ে গেলে wrap হয়ে 0 থেকে শুরু হয়

সমস্যা:
  নতুন XID পুরনো committed rows এর চেয়ে "আগে" হয়ে যায়
  PostgreSQL বুঝতে পারে না কোনটা visible
  → PostgreSQL SHUTDOWN করে database protect করতে!

Prevention:
  VACUUM FREEZE পুরনো rows এর xmin কে "frozen" করে
  Frozen row সবার কাছে সবসময় visible (XID compare দরকার নেই)
  Autovacuum নিয়মিত FREEZE করে
```

```sql
-- XID age দেখো — বড় হলে বিপদ!
SELECT datname,
    age(datfrozenxid) AS xid_age,
    2000000000 - age(datfrozenxid) AS safe_remaining
FROM pg_database
ORDER BY xid_age DESC;
-- safe_remaining < 500 million → জরুরি VACUUM FREEZE দরকার!

-- Table এর dead tuple দেখো
SELECT schemaname, relname,
    n_live_tup, n_dead_tup,
    ROUND(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct,
    last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 10;

-- Manual VACUUM
VACUUM VERBOSE ANALYZE users;        -- Verbose output সহ
VACUUM FREEZE users;                  -- XID freeze করো
```

### Autovacuum Tuning

```ini
# postgresql.conf
autovacuum                       = on      # কখনো বন্ধ করো না!
autovacuum_naptime               = 1min    # কতক্ষণ পর পর check করবে
autovacuum_max_workers           = 5       # একসাথে কতটা worker চলবে

# Trigger threshold: dead_tuples > threshold + (scale_factor × live_tuples)
autovacuum_vacuum_threshold      = 50      # Minimum 50 dead tuple
autovacuum_vacuum_scale_factor   = 0.02   # বা table এর 2%

autovacuum_analyze_threshold     = 50
autovacuum_analyze_scale_factor  = 0.05

autovacuum_vacuum_cost_delay     = 2ms    # Throttle (IO burst কমাতে)

# Specific table এর জন্য override (বড় table তে)
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.01,
    autovacuum_vacuum_threshold    = 100
);
```

---

## 1.9 Complete Read Path (Full Cycle)

**Query:** `SELECT * FROM users WHERE id = 5`

```
Step 1: Client → Postmaster
  TCP connection (port 5432)
  Postmaster: pg_hba.conf check → fork() → নতুন Backend Process

Step 2: Backend — Parser
  "SELECT * FROM users WHERE id = 5"
  Parse Tree তৈরি:
    SELECT → columns: * (all)
    FROM   → relation: users
    WHERE  → id = 5
  Syntax ভুল থাকলে এখানেই error

Step 3: Backend — Rewriter
  View আছে? → Underlying table এ expand করো
  Rule আছে? → Apply করো

Step 4: Backend — Planner/Optimizer
  pg_statistic থেকে statistics দেখে:
    - users table এ কতটা row
    - id column এর cardinality
    - id column এ index আছে? → users_pkey (B-Tree)

  Cost calculation:
    Plan A: Seq Scan → cost=0..150 (সব row scan)
    Plan B: Index Scan on users_pkey → cost=0.29..8.30 (1 row)
    Winner: Plan B (কম cost)

Step 5: Executor → Storage Manager
  "id=5 এর page দাও"

Step 6: Shared Buffers Check
  ┌────────────────────────────────────────────────┐
  │ এই page কি Shared Buffers এ আছে? (buffer hit?) │
  │                                                │
  │  YES → page থেকে tuple read করো (disk I/O নেই) │
  │                                                │
  │  NO  → OS Page Cache এ আছে?                    │
  │          YES → OS cache থেকে load              │
  │          NO  → Disk থেকে read (slowest)        │
  │        Shared Buffers এ load করো               │
  └────────────────────────────────────────────────┘

Step 7: MVCC Visibility Check
  Row এর xmin দেখো → CLOG (pg_xact) এ check → committed?
  Row এর xmax দেখো → 0? → not deleted → visible
  Current transaction এর snapshot এর সাথে মেলাও
  → Visible হলে return করো

Step 8: Result → Client
  Tuple → Executor → Result set → Network buffer → Client

READ এর সময় component states:
  Shared Buffers  : Page এর usage count বাড়ে
  CLOG            : Read হয় (transaction status check)
  WAL             : কিছুই হয় না (read এ WAL লেখা হয় না)
  Locks           : AccessShareLock on table (weakest lock)
```

---

## 1.10 Complete Write Path (Full Cycle)

**Query:** `INSERT INTO users (name, email) VALUES ('Alice', 'alice@example.com')`

```
Step 1-4: Connection → Parse → Rewrite → Plan
  Write permission check হয়

Step 5: Transaction ID (XID) Assign
  XID = globally unique, monotonically increasing number
  এই transaction এর "identity"

Step 6: Free Space Map (FSM) দেখো
  কোন page এ নতুন row রাখার space আছে?
  FSM একটা tree structure যেটা প্রতিটা page এর free space track করে

Step 7: Target Page Shared Buffers এ আনো
  Page আছে? → সরাসরি
  নেই? → Disk থেকে load

Step 8: Page এ Tuple Insert করো
  নতুন tuple:
    xmin = current XID   (আমি insert করেছি)
    xmax = 0             (কেউ delete করেনি)
    ctid = (page, slot)  (physical location)
    data = 'Alice', 'alice@...'
  Page state: Clean → Dirty

Step 9: Index Update
  users_pkey (PRIMARY KEY) B-Tree এ নতুন entry
  অন্য সব index এও entry

Step 10: WAL Record লেখো
  WAL Buffers এ:
    "Page X এ এই bytes insert হয়েছে"
  এখনো disk এ নেই

Step 11: COMMIT
  ① WAL Buffers → pg_wal/ disk flush (synchronous_commit=on)
     এই মুহূর্তে transaction durable হলো
     Crash হলেও WAL থেকে recover করা যাবে
  ② CLOG (pg_xact) এ XID = COMMITTED mark
  ③ Lock release
  ④ Binary Log → WAL Sender → Replica (replication চললে)

NOTE: COMMIT হওয়ার পরেও:
  - Data page (Shared Buffers) এখনো dirty
  - WAL disk এ আছে → crash হলেও safe
  - Background Writer বা Checkpointer পরে data page disk এ লিখবে

WRITE এর সময় component states:
  Shared Buffers  : Target page dirty হয়
  FSM             : Space usage update হয়
  WAL Buffers     : Change record লেখা হয়
  WAL (disk)      : COMMIT এ flush হয়
  CLOG            : XID = COMMITTED
  Locks           : RowExclusiveLock (INSERT/UPDATE/DELETE)
```

---

## 1.11 UPDATE Operation — PostgreSQL এ Update মানে Delete + Insert

এটা MySQL থেকে সম্পূর্ণ আলাদা। MySQL UPDATE in-place হয়। PostgreSQL এ UPDATE মানে পুরনো row কে "dead" করে নতুন row insert করা।

```
Query: UPDATE users SET salary = 50000 WHERE id = 5

Before:
  (xmin=100, xmax=0, id=5, salary=40000, ctid=(0,1))

Step 1: id=5 এর row খোঁজো, RowExclusiveLock নাও

Step 2: পুরনো tuple "delete" করো (soft delete):
  xmax = current XID (e.g., 200)
  Old row: (xmin=100, xmax=200, id=5, salary=40000) ← DEAD

Step 3: নতুন tuple insert করো:
  New row: (xmin=200, xmax=0, id=5, salary=50000, ctid=(0,5)) ← LIVE

Step 4: HOT Update (Heap Only Tuple) সম্ভব হলে:
  Condition: update করা column এ কোনো index নেই
             AND নতুন tuple একই page এ জায়গা পায়
  তাহলে: Index update করতে হয় না
          পুরনো tuple ctid → নতুন tuple point করে
          → Fast update (index I/O নেই)
  
  HOT সম্ভব না হলে: সব index এ নতুন entry, পুরনো entry ও থাকে

Step 5: WAL, COMMIT

After:
  Page এ দুটো tuple (old dead + new live)
  VACUUM পরে dead tuple সরাবে
  → Autovacuum না চললে page তে dead tuple জমতে থাকবে = BLOAT
```

---

## 1.12 DELETE Operation

```
Query: DELETE FROM users WHERE id = 5

Step 1: Row খোঁজো, RowExclusiveLock নাও

Step 2: Soft Delete:
  tuple এর xmax = current XID
  Physically কিছু মুছে না!
  Row আছে, শুধু "এই XID delete করেছে" mark হলো

কেন soft delete?
  অন্য transaction যারা আগে শুরু হয়েছে তারা এখনো এই row দেখতে পারে (MVCC)
  Rollback হলে শুধু xmax clear করলেই row আবার visible

Step 3: Index entries এখনো আছে
  Index → Heap এ dead tuple কে point করছে
  VACUUM পরে index entry এবং heap tuple দুটোই সরাবে

Step 4: WAL, COMMIT

Purge (VACUUM এর কাজ):
  সব active transaction এর snapshot এর আগের committed dead tuples physically মুছে
  Index এর dead entries সরায়
  Disk space free করে FSM এ return করে
```

---

## 1.13 ROLLBACK Operation

```
ROLLBACK বা error এ:

Method: CLOG update — physical undo করতে হয় না!

Step 1: CLOG এ XID = ABORTED mark করো
  (MySQL এ Undo Log থেকে physically reverse করতে হয়)
  (PostgreSQL এ শুধু CLOG update করলেই হয়)

Step 2: Lock release

Step 3: VACUUM পরে cleanup:
  ABORTED XID এর tuples → VACUUM সরিয়ে দেবে
  (কেউ আর এই tuples visible হিসেবে দেখবে না, CLOG বলে ABORTED)

কেন এত সহজ?
  MVCC visibility check এ CLOG দেখা হয়
  XID ABORTED → row automatically invisible সবার কাছে
  Physical undo দরকার নেই
```

---

## 1.14 Crash Recovery

```
Server হঠাৎ বন্ধ হয়ে গেলে restart এ কী হয়:

Step 1: pg_control file পড়ো
  Last checkpoint LSN জানো
  Cluster state: in_crash_recovery

Step 2: WAL Redo Phase
  Last checkpoint LSN থেকে শুরু করো
  WAL এর প্রতিটা record reapply করো
  Committed transactions এর changes পুনরায় apply

  উদাহরণ:
    WAL record: "XID 200 এ Page 5 এ id=5 salary=50000 হয়েছে"
    CLOG check: XID 200 COMMITTED? → হ্যাঁ
    → Page 5 এ salary=50000 apply করো

Step 3: Incomplete Transactions
  CLOG এ IN_PROGRESS থাকা XID গুলো ABORTED mark করো
  VACUUM পরে এদের tuples সরাবে
  Physical undo দরকার নেই (MySQL এর মতো)

Step 4: Normal operation শুরু

Recovery speed নির্ভর করে:
  Last checkpoint থেকে crash পর্যন্ত কতটা WAL জমেছে তার উপর
  checkpoint_timeout ছোট → কম WAL replay → faster recovery
  কিন্তু ছোট করলে বেশি I/O → tradeoff
```

---

## 1.15 Concurrent Read + Write (MVCC in Action)

```
Timeline:
  T=1: Transaction A শুরু (XID=100)
       Snapshot: "XID 99 এর আগে যা committed তা দেখবো"
       SELECT salary FROM users WHERE id=5 → 40000

  T=2: Transaction B শুরু (XID=200)
       UPDATE users SET salary=50000 WHERE id=5
       Old tuple: xmin=50, xmax=200
       New tuple: xmin=200, xmax=0
       COMMIT → CLOG: XID 200 = COMMITTED

  T=3: Transaction A আবার SELECT (READ COMMITTED vs REPEATABLE READ)

READ COMMITTED (default):
  A এর snapshot প্রতিটা statement এ নতুন নেওয়া হয়
  New tuple: xmin=200, CLOG → 200 COMMITTED → visible!
  → A দেখবে salary=50000 (B এর change দেখে)

REPEATABLE READ:
  A এর snapshot transaction শুরুতে নেওয়া হয়েছিল (XID 99)
  New tuple: xmin=200 > 99 → NOT in A's snapshot → not visible
  Old tuple: xmin=50 (<=99), xmax=200 (>99) → VISIBLE
  → A দেখবে salary=40000 (transaction শুরুর মতো)

ফলাফল:
  B এর write A এর read কে block করেনি
  A এর read B এর write কে block করেনি
  → High concurrency, no read-write contention
```

---

## 1.16 Component Summary

| Component | MySQL Equivalent | কাজ |
|---|---|---|
| Shared Buffers | Buffer Pool | Data page cache |
| WAL Buffers | Log Buffer | WAL memory buffer |
| pg_wal/ | ib_logfile* (Redo Log) | Crash recovery, replication |
| pg_xact/ (CLOG) | InnoDB trx system | Transaction commit status |
| FSM (Free Space Map) | InnoDB free space | INSERT এর জন্য space খোঁজা |
| Visibility Map | — | VACUUM optimization, Index Only Scan |
| Backend Process | Thread | Per-connection query processing |
| Postmaster | MySQL mysqld main thread | Master process, fork করে |
| Checkpointer | InnoDB Master Thread (부분) | Dirty pages flush |
| Background Writer | InnoDB Page Cleaner | Dirty page flush (smooth I/O) |
| Autovacuum | InnoDB Purge Thread | Dead tuple cleanup |
| WAL Sender | MySQL Binlog Dump Thread | Replication stream |
| WAL Receiver | MySQL IO Thread | Replication receive |
| Archiver | — | WAL archive for PITR |
| pg_hba.conf | mysql.user @host column | Authentication rules |
| VACUUM | InnoDB Purge (automatic) | Space reclaim (manual/auto) |

---

# 2. Configuration & Parameters

## 2.1 Configuration Files — তিনটা গুরুত্বপূর্ণ File

### postgresql.conf — Main Configuration

```ini
# সব parameter এখানে
shared_buffers = 4GB
max_connections = 200
...
```

### postgresql.auto.conf — ALTER SYSTEM এর Changes

```sql
-- ALTER SYSTEM দিয়ে change করলে এই file এ লেখা হয়
ALTER SYSTEM SET work_mem = '64MB';
-- postgresql.auto.conf এ override হয় — restart ছাড়া কাজ করে না
-- postgresql.conf এর চেয়ে বেশি priority
```

### pg_hba.conf — Authentication Rules

```
কে (user) কোথা থেকে (IP/host) কোন database তে কীভাবে (method) connect করতে পারবে
MySQL এর মতো user definition এ host লেখা হয় না
এখানে আলাদাভাবে rules লেখা হয়
```

```sql
-- Config file এর location দেখো
SHOW config_file;
SHOW hba_file;
SHOW data_directory;

-- Reload (restart ছাড়া — sighup context parameter এর জন্য)
SELECT pg_reload_conf();
-- অথবা: sudo systemctl reload postgresql-16

-- কোন parameter কোন context এ পড়ো
SELECT name, setting, unit, context, short_desc
FROM pg_settings
WHERE name IN ('shared_buffers', 'work_mem', 'max_connections', 'synchronous_commit');
```

**`context` মানে:**
| Context | পরিবর্তন করতে | Example |
|---|---|---|
| `internal` | Compile-time, পরিবর্তন সম্ভব না | `block_size` |
| `postmaster` | Restart দরকার | `shared_buffers`, `max_connections` |
| `sighup` | Reload যথেষ্ট | `log_min_duration_statement` |
| `superuser` | Superuser SET করতে পারে | `work_mem` |
| `user` | যেকোনো user SET করতে পারে | `search_path` |

---

## 2.2 Memory Parameters

```ini
# postgresql.conf

# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# সবচেয়ে গুরুত্বপূর্ণ — restart দরকার
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

shared_buffers = 4GB
# RAM এর 25% দাও
# MySQL এর মতো 70-80% না — OS Page Cache ও কাজ করে
# 8GB RAM  → 2GB
# 16GB RAM → 4GB
# 32GB RAM → 8GB

max_connections = 200
# প্রতিটা connection ~5-10MB shared memory নেয়
# 200 × 10MB = 2GB শুধু connections এ!
# বেশি connection দরকার হলে pgBouncer ব্যবহার করো

# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# গুরুত্বপূর্ণ — reload যথেষ্ট
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

work_mem = 64MB
# প্রতিটা sort, hash join operation এ এতটুকু memory পায়
# ⚠️ এটা per-operation, per-query নয়!
# একটা complex query তে 10টা sort operation থাকতে পারে
# 200 connections × 10 operations × 64MB = 128GB!
# Production এ conservative রাখো (4-64MB)
# Specific session এ বাড়াও দরকার হলে:
# SET work_mem = '256MB';  (session level)

maintenance_work_mem = 1GB
# VACUUM, CREATE INDEX, REINDEX এর জন্য
# এটা কম connections এ চলে, বেশি দেওয়া safe
# বড় দিলে VACUUM এবং index build fast হয়

effective_cache_size = 12GB
# RAM এর 75% দাও
# Actual memory allocate করে না — শুধু Optimizer কে hint দেয়
# "OS cache তে এতটুকু available থাকবে"
# Optimizer এই hint দিয়ে Index Scan vs Seq Scan decide করে

wal_buffers = 64MB
# Default: shared_buffers এর 1/32 (auto)
# Write-heavy workload এ 64MB explicit দাও
# Restart দরকার

temp_buffers = 8MB
# Per-session temporary table buffer
# Temporary table বেশি ব্যবহার হলে বাড়াও
```

**Memory Sizing Example (16GB RAM):**
```
shared_buffers       = 4GB    (25%)
effective_cache_size = 12GB   (75%)
work_mem             = 64MB   (conservative)
maintenance_work_mem = 1GB
wal_buffers          = 64MB
OS এর জন্য          = ~4GB বাকি থাকে
```

**Hands-on: Memory Parameter Apply করো এবং Verify করো**

```bash
# Step 1: Current values দেখো
psql -U postgres -c "SHOW shared_buffers; SHOW work_mem; SHOW effective_cache_size;"

# Step 2: postgresql.conf এ change করো
sudo nano /var/lib/pgsql/16/data/postgresql.conf
# অথবা ALTER SYSTEM দিয়ে:
psql -U postgres -c "ALTER SYSTEM SET shared_buffers = '4GB';"
psql -U postgres -c "ALTER SYSTEM SET work_mem = '64MB';"
psql -U postgres -c "ALTER SYSTEM SET maintenance_work_mem = '1GB';"
psql -U postgres -c "ALTER SYSTEM SET effective_cache_size = '12GB';"

# Step 3: Restart (shared_buffers restart দরকার)
sudo systemctl restart postgresql-16

# Step 4: Verify
psql -U postgres -c "SHOW shared_buffers;"
psql -U postgres -c "SHOW work_mem;"

# Step 5: Shared memory actually allocate হয়েছে কিনা OS level এ দেখো
ipcs -m | grep postgres
# অথবা:
cat /proc/$(pgrep -f "postgres: postmaster")/status | grep -i vmrss
```

```sql
-- সব memory-related settings একসাথে দেখো
SELECT name, setting, unit, context,
    CASE context
        WHEN 'postmaster' THEN 'restart needed'
        WHEN 'sighup'     THEN 'reload enough'
        WHEN 'user'       THEN 'session level ok'
    END AS how_to_apply
FROM pg_settings
WHERE name IN (
    'shared_buffers','work_mem','maintenance_work_mem',
    'effective_cache_size','wal_buffers','temp_buffers',
    'max_connections'
)
ORDER BY name;

-- Buffer hit rate verify করো (change এর পরে ভালো হওয়া উচিত)
SELECT
    ROUND(
        sum(heap_blks_hit) * 100.0 /
        NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0),
    2) AS buffer_hit_rate_pct
FROM pg_statio_user_tables;
-- 99%+ হলে shared_buffers adequate

-- work_mem কম হলে temp files দেখা যাবে
SELECT datname, temp_files, pg_size_pretty(temp_bytes) AS temp_size
FROM pg_stat_database
WHERE datname = current_database();
-- temp_files > 0 এবং temp_size বড় হলে work_mem বাড়াও
```

---

## 2.3 WAL Parameters

```ini
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# WAL Level — restart দরকার
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

wal_level = replica
# minimal  → Crash recovery শুধু, replication সম্ভব না
# replica  → Streaming replication সম্ভব (Production default)
# logical  → Logical replication সম্ভব (replica এর সব + বেশি info)

# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# Durability — reload যথেষ্ট
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

synchronous_commit = on
# on = COMMIT এর আগে WAL disk এ flush → safe, slow
# off = flush এর আগেই ack → fast, শেষ কিছু transaction হারাতে পারে

# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# Checkpoint — reload যথেষ্ট
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

checkpoint_timeout = 5min
# Maximum কতক্ষণ পর পর checkpoint হবে
# ছোট → faster crash recovery, বেশি I/O
# বড় → slower recovery, কম I/O

max_wal_size = 2GB
# এই পরিমাণ WAL জমলে checkpoint trigger হয়
# বড় দিলে checkpoint কম → I/O কম, কিন্তু recovery slow
# Production: 1-4GB

min_wal_size = 256MB
# Minimum WAL size retain করো (recycle করার জন্য)

wal_compression = on
# WAL compress করে → disk এবং network I/O কমে
# CPU সামান্য বেশি লাগে — modern hardware এ worth it

# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# Archiving (PITR এর জন্য) — reload যথেষ্ট
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

archive_mode = on
archive_command = 'pgbackrest --stanza=main archive-push %p'
# %p = WAL file এর full path
# প্রতিটা completed WAL segment এই command দিয়ে archive হয়

archive_timeout = 300
# 5 মিনিটে একবার force archive (idle server এও WAL archive হবে)
# PITR এর gap কমায়
```

**Hands-on: WAL Settings Verify করো**

```bash
# WAL level এবং archiving status দেখো
psql -U postgres << 'EOF'
SHOW wal_level;
SHOW archive_mode;
SHOW archive_command;
SHOW synchronous_commit;
SHOW checkpoint_timeout;
SHOW max_wal_size;
EOF

# Archive হচ্ছে কিনা দেখো
psql -U postgres -c "SELECT * FROM pg_stat_archiver;"
# last_archived_wal: সর্বশেষ archive হওয়া WAL file
# failed_count: 0 হওয়া উচিত

# Manual WAL switch করো — archive test করো
psql -U postgres -c "SELECT pg_switch_wal();"
# এখন archive directory তে নতুন file দেখা উচিত
ls -lt /archive/wal/ | head -5

# Current WAL position এবং size দেখো
psql -U postgres -c "SELECT pg_current_wal_lsn(), pg_walfile_name(pg_current_wal_lsn());"

# Checkpoint কতক্ষণ পর পর হচ্ছে — log দেখো
sudo grep "checkpoint" /var/lib/pgsql/16/data/log/postgresql-$(date +%Y-%m-%d).log | tail -10
# "checkpoint complete" দেখলে checkpoint সফলভাবে হচ্ছে
# "checkpoint request" বেশি দেখলে → max_wal_size বাড়াও
```

---

## 2.4 Connection Parameters

```ini
listen_addresses = '*'
# '*' = সব network interface এ listen
# '172.16.93.140' = specific IP only (recommended)
# 'localhost' = local connections only

port = 5432                 # Default PostgreSQL port

# Keepalive (idle connection detect করতে)
tcp_keepalives_idle     = 60   # 60s idle এর পর keepalive পাঠাও
tcp_keepalives_interval = 10   # প্রতি 10s এ retry
tcp_keepalives_count    = 6    # 6 বার fail হলে connection বন্ধ

# SSL
ssl          = on
ssl_cert_file = 'server.crt'
ssl_key_file  = 'server.key'
```

**Hands-on: Connection Verify করো**

```bash
# কোন IP তে listen করছে
psql -U postgres -c "SHOW listen_addresses;"
ss -tnlp | grep 5432

# Max connections দেখো এবং current usage
psql -U postgres -c "
SELECT
    (SELECT setting::int FROM pg_settings WHERE name='max_connections') AS max_conn,
    COUNT(*) AS current_conn,
    ROUND(COUNT(*) * 100.0 / (SELECT setting::int FROM pg_settings WHERE name='max_connections'), 1) AS used_pct
FROM pg_stat_activity;"

# Remote connection test (অন্য machine থেকে)
psql -h 172.16.93.140 -U postgres -d mydb -c "SELECT inet_server_addr(), inet_server_port();"

# SSL active কিনা দেখো
psql -U postgres -c "SELECT ssl, version, cipher FROM pg_stat_ssl WHERE pid = pg_backend_pid();"
```

---

## 2.5 Query Planner Parameters

```ini
# Hardware অনুযায়ী tune করো — planner cost model

random_page_cost = 1.1
# SSD: 1.1 (sequential এর মাত্র 10% বেশি)
# HDD: 4.0 (random seek অনেক expensive)
# Cloud SSD: 1.1-1.5

seq_page_cost = 1.0          # Baseline (পরিবর্তন করো না সাধারণত)

effective_io_concurrency = 200
# SSD: 200 (parallel I/O capable)
# HDD: 2-4
# Cloud SSD: 100-300

# Statistics — Planner কতটা accurate statistics রাখবে
default_statistics_target = 100
# Default 100 (100 values sample)
# High cardinality column এ বাড়াও:
# ALTER TABLE orders ALTER COLUMN user_id SET STATISTICS 500;

# Parallel Query
max_parallel_workers_per_gather = 4   # Query তে maximum 4 parallel worker
max_parallel_workers            = 8   # Server এ maximum 8 parallel worker

# Partition-wise operations (Partitioned table এ performance boost)
enable_partitionwise_join       = on  # Partition কে partition এর সাথে join (default off)
enable_partitionwise_aggregate  = on  # Partition এ separately aggregate (default off)
# এই দুটো on করলে partitioned table এ query অনেক faster হতে পারে
# কিন্তু planning time বাড়ে — partition বেশি হলে tradeoff দেখো
```

**Hands-on: Planner Settings Verify এবং Impact দেখো**

```sql
-- Current planner settings
SELECT name, setting, unit
FROM pg_settings
WHERE name IN (
    'random_page_cost','seq_page_cost','effective_io_concurrency',
    'default_statistics_target','max_parallel_workers_per_gather',
    'max_parallel_workers','jit'
);

-- SSD আছে কিনা OS level এ confirm করো
-- (bash এ চালাও):
-- cat /sys/block/sda/queue/rotational
-- 0 = SSD, 1 = HDD

-- Planner এর cost estimate দেখো
EXPLAIN SELECT * FROM orders WHERE user_id = 5;
-- cost=0.00..8.30 rows=1 width=40
-- random_page_cost কমালে (SSD) Index Scan আরো prefer হবে

-- Statistics target impact দেখো
SELECT attname, n_distinct, correlation
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'user_id';
-- correlation: 1.0 = perfectly sorted, 0 = random
-- n_distinct: -1 = unique, positive number = distinct count estimate

-- Parallel query আসলে চলছে কিনা দেখো
EXPLAIN (ANALYZE, VERBOSE) SELECT COUNT(*) FROM large_table;
-- "Workers Planned: 4" এবং "Workers Launched: 4" দেখলে parallel চলছে

-- Parallel force করে test করো
SET max_parallel_workers_per_gather = 4;
EXPLAIN (ANALYZE) SELECT COUNT(*), SUM(total) FROM orders;
-- Gather node দেখাবে parallel চলছে কিনা
RESET max_parallel_workers_per_gather;
```

---

## 2.6 Logging Parameters

```ini
# Log destination
logging_collector = on              # File এ log collect করো
log_directory     = 'log'           # $PGDATA/log/
log_filename      = 'postgresql-%Y-%m-%d.log'
log_rotation_age  = 1d
log_rotation_size = 100MB

# কী log করবো
log_min_duration_statement = 1000
# 1000ms = 1 second এর বেশি লাগলে log করো
# MySQL এর slow_query_log + long_query_time এর মতো
# Production শুরুতে 1000ms, পরে tune করো

log_line_prefix = '%m [%p] %q%u@%d '
# %m = timestamp with milliseconds
# %p = process ID
# %u = username
# %d = database name
# Example: 2024-03-08 14:30:05.123 [12345] postgres@mydb

log_connections    = on     # নতুন connection log
log_disconnections = on     # Disconnect log
log_lock_waits     = on     # Lock wait log (important for debugging)
log_temp_files     = 0      # সব temporary file log করো (0=all)
log_checkpoints    = on     # Checkpoint log
log_autovacuum_min_duration = 250ms   # 250ms+ autovacuum log করো

# Debug (production এ off রাখো)
log_statement = 'none'      # none/ddl/mod/all
# 'all' দিলে সব query log → disk I/O অনেক বাড়ে
```

**Hands-on: Logging Setup এবং Slow Query Test করো**

```bash
# Step 1: Logging চালু আছে কিনা verify করো
psql -U postgres -c "SHOW logging_collector; SHOW log_directory; SHOW log_min_duration_statement;"

# Step 2: Log file কোথায় আছে দেখো
psql -U postgres -c "SELECT pg_current_logfile();"
# Result: log/postgresql-2024-03-08_000000.log

# Step 3: Log directory দেখো
ls -lh /var/lib/pgsql/16/data/log/
sudo tail -20 /var/lib/pgsql/16/data/log/postgresql-$(date +%Y-%m-%d)*.log

# Step 4: Slow query test করো
# log_min_duration_statement = 0 সেট করো সব query log করতে (test only!)
psql -U postgres -c "ALTER SYSTEM SET log_min_duration_statement = 0;"
psql -U postgres -c "SELECT pg_reload_conf();"

# একটা slow query চালাও
psql -U postgres -c "SELECT pg_sleep(2);"

# Log এ দেখো — duration line দেখাবে
sudo tail -5 /var/lib/pgsql/16/data/log/postgresql-$(date +%Y-%m-%d)*.log
# দেখবে: duration: 2000.xxx ms  statement: SELECT pg_sleep(2);

# Step 5: Production value restore করো
psql -U postgres -c "ALTER SYSTEM SET log_min_duration_statement = 1000;"
psql -U postgres -c "SELECT pg_reload_conf();"

# Step 6: Log rotation manually করো
psql -U postgres -c "SELECT pg_rotate_logfile();"
ls -lh /var/lib/pgsql/16/data/log/
```

```sql
-- Log settings সব একসাথে দেখো
SELECT name, setting, unit
FROM pg_settings
WHERE name LIKE 'log%'
ORDER BY name;

-- Slow queries দেখো pg_stat_statements থেকে
SELECT LEFT(query, 80) AS query, calls,
    ROUND(mean_exec_time::numeric, 2) AS avg_ms
FROM pg_stat_statements
WHERE mean_exec_time > 1000   -- 1 second এর বেশি
ORDER BY mean_exec_time DESC
LIMIT 10;
```

---

**Hands-on: Logging Setup এবং Slow Query ধরো**

```bash
# Step 1: Logging parameters set করো
psql -U postgres << 'EOF'
ALTER SYSTEM SET logging_collector = on;
ALTER SYSTEM SET log_min_duration_statement = 1000;
ALTER SYSTEM SET log_checkpoints = on;
ALTER SYSTEM SET log_lock_waits = on;
ALTER SYSTEM SET log_connections = on;
ALTER SYSTEM SET log_line_prefix = '%m [%p] %q%u@%d ';
EOF

# logging_collector = postmaster context → restart দরকার
sudo systemctl restart postgresql-16

# Step 2: Log directory confirm করো
psql -U postgres -c "SHOW log_directory;"
ls -la /var/lib/pgsql/16/data/log/

# Step 3: Intentionally slow query চালাও — log এ catch হয় কিনা দেখো
psql -U postgres -d mydb -c "SELECT pg_sleep(2);"
# 2 seconds → log_min_duration_statement=1000ms এর বেশি → log হবে

# Step 4: Log তে দেখো
sudo tail -20 /var/lib/pgsql/16/data/log/postgresql-$(date +%Y-%m-%d).log
# Output:
# 2024-03-08 14:30:07.123 [12345] postgres@mydb
#   duration: 2003.456 ms  statement: SELECT pg_sleep(2);

# Step 5: Slowest queries summary করো
sudo grep "duration:" /var/lib/pgsql/16/data/log/postgresql-$(date +%Y-%m-%d).log | \
    awk '{print $8, $0}' | sort -rn | head -10

# Step 6: Runtime এ threshold change করো (reload যথেষ্ট, restart লাগে না)
psql -U postgres -c "ALTER SYSTEM SET log_min_duration_statement = 500;"
psql -U postgres -c "SELECT pg_reload_conf();"
psql -U postgres -c "SHOW log_min_duration_statement;"  -- 500ms confirm
```

```sql
-- pg_stat_statements দিয়ে top slow queries দেখো
-- (shared_preload_libraries = 'pg_stat_statements' দরকার)
SELECT
    LEFT(query, 80)                            AS query,
    calls,
    ROUND(mean_exec_time::numeric, 2)          AS avg_ms,
    ROUND(total_exec_time::numeric, 2)         AS total_ms,
    ROUND(stddev_exec_time::numeric, 2)        AS stddev_ms
FROM pg_stat_statements
WHERE mean_exec_time > 1000
ORDER BY total_exec_time DESC
LIMIT 10;
```


## 2.7 Replication Parameters

```ini
# Primary Server এ

wal_level            = replica       # Minimum replica চাই
max_wal_senders      = 10            # Maximum কতটা Replica connect করতে পারবে
wal_keep_size        = 1GB           # Replica এর জন্য WAL retain করো
max_replication_slots = 10           # Replication slot maximum
hot_standby          = on            # Replica তে read query allow
synchronous_standby_names = ''       # Async ('' = কোনো specific standby না)

# Standby Server এ

hot_standby          = on            # Read queries allow করো
hot_standby_feedback = on
# Replica query চলার সময় Primary এ autovacuum সেই rows মুছবে না
# Long-running queries on Replica → Primary এ "query conflict" এড়ায়

# Standby Conflict Resolution — কীভাবে কাজ করে:
# Primary: UPDATE/DELETE চলছে → WAL পাঠায় Replica তে
# Replica: চলমান SELECT ঐ rows এর পুরনো version চাইছে → conflict!
# PostgreSQL কে decide করতে হবে: query cancel করবে নাকি WAL apply delay করবে

max_standby_streaming_delay = 30s
# Default: 30s
# Replica তে streaming WAL apply করার আগে maximum কতক্ষণ অপেক্ষা করবে
# যদি conflict হয় এবং query ঐ সময়ের মধ্যে শেষ না হয় → query cancel হয়
# -1 = unlimited wait (query শেষ না হওয়া পর্যন্ত WAL apply delay হবে)
# Production: hot_standby_feedback=on থাকলে conflict কম হয়
# Report/Analytics Replica: -1 (long queries যেন cancel না হয়)
# Low-latency Replica:     30s (default, replica lag বেশি হবে না)

max_standby_archive_delay = 30s
# Archive থেকে recovery তে (PITR) একই concept

wal_receiver_timeout     = 60s       # Primary এর সাথে connection timeout
recovery_min_apply_delay = 0         # Delayed standby (0 = no delay)
# '30min' → 30 মিনিট আগের data apply করবে (accidental delete protection)
```

**Hands-on: Replication Parameters Apply এবং Verify করো**

```bash
# Primary তে — Step 1: Parameters set করো
sudo nano /var/lib/pgsql/16/data/postgresql.conf
# wal_level = replica
# max_wal_senders = 10
# wal_keep_size = 1GB
# hot_standby = on

sudo systemctl restart postgresql-16

# Primary তে — Step 2: Settings verify করো
psql -U postgres << 'EOF'
SHOW wal_level;
SHOW max_wal_senders;
SHOW wal_keep_size;
SHOW hot_standby;
SELECT pg_current_wal_lsn();
EOF
```

```sql
-- Primary তে — WAL sender slots কতটা used
SELECT count(*) AS wal_senders_active,
    (SELECT setting::int FROM pg_settings WHERE name = 'max_wal_senders') AS max_senders
FROM pg_stat_replication;

-- Replica তে — connection এবং lag verify করো
SELECT
    pg_is_in_recovery()                              AS is_standby,
    pg_last_wal_receive_lsn()                        AS received_lsn,
    pg_last_wal_replay_lsn()                         AS replayed_lsn,
    now() - pg_last_xact_replay_timestamp()          AS replication_lag,
    pg_last_xact_replay_timestamp()                  AS last_replayed_at;

-- hot_standby_feedback কাজ করছে কিনা দেখো
-- Replica তে long query চালাও, Primary তে autovacuum conflict দেখো:
SELECT query, wait_event_type, wait_event
FROM pg_stat_activity
WHERE wait_event_type = 'Lock'
   OR wait_event = 'recovery conflict';
```

---

## 2.8 ALTER SYSTEM — Persistent Runtime Change

```sql
-- postgresql.auto.conf এ লেখে, restart/reload এ active হয়
ALTER SYSTEM SET shared_buffers = '4GB';       -- restart দরকার
ALTER SYSTEM SET work_mem       = '64MB';      -- reload যথেষ্ট
ALTER SYSTEM SET log_min_duration_statement = 500;

-- Reload (sighup context parameter)
SELECT pg_reload_conf();

-- Restart required check
SELECT name, setting, pending_restart
FROM pg_settings
WHERE pending_restart = true;   -- restart এর আগ পর্যন্ত পুরনো value চলবে

-- Reset
ALTER SYSTEM RESET work_mem;
ALTER SYSTEM RESET ALL;

-- Current effective value
SHOW work_mem;
SELECT current_setting('work_mem');
```

---

## 2.9 Production postgresql.conf (16GB RAM, SSD)

```ini
#─────────────────────────────────
# Connection
#─────────────────────────────────
listen_addresses         = '*'
port                     = 5432
max_connections          = 200       # pgBouncer দিয়ে আরো বাড়ানো যাবে

#─────────────────────────────────
# Memory
#─────────────────────────────────
shared_buffers           = 4GB       # RAM 25%
work_mem                 = 64MB      # Conservative, session এ বাড়াও
maintenance_work_mem     = 1GB
effective_cache_size     = 12GB      # RAM 75%
wal_buffers              = 64MB

#─────────────────────────────────
# WAL & Checkpoint
#─────────────────────────────────
wal_level                = replica
synchronous_commit       = on
checkpoint_timeout       = 5min
max_wal_size             = 2GB
min_wal_size             = 256MB
wal_compression          = on

#─────────────────────────────────
# Archiving (PITR)
#─────────────────────────────────
archive_mode             = on
archive_command          = 'pgbackrest --stanza=main archive-push %p'
archive_timeout          = 300

#─────────────────────────────────
# Replication
#─────────────────────────────────
max_wal_senders          = 10
max_replication_slots    = 10
wal_keep_size            = 1GB
hot_standby              = on
hot_standby_feedback     = on

#─────────────────────────────────
# Query Planner (SSD)
#─────────────────────────────────
random_page_cost         = 1.1
effective_io_concurrency = 200
default_statistics_target = 100

#─────────────────────────────────
# Parallel Query
#─────────────────────────────────
max_parallel_workers_per_gather = 4
max_parallel_workers            = 8

#─────────────────────────────────
# Logging
#─────────────────────────────────
logging_collector        = on
log_directory            = 'log'
log_filename             = 'postgresql-%Y-%m-%d.log'
log_rotation_age         = 1d
log_min_duration_statement = 1000
log_checkpoints          = on
log_lock_waits           = on
log_autovacuum_min_duration = 250ms
log_line_prefix          = '%m [%p] %q%u@%d '
log_connections          = on
log_disconnections       = on

#─────────────────────────────────
# Autovacuum
#─────────────────────────────────
autovacuum                       = on
autovacuum_max_workers           = 5
autovacuum_naptime               = 30s
autovacuum_vacuum_scale_factor   = 0.02
autovacuum_analyze_scale_factor  = 0.01
autovacuum_vacuum_cost_delay     = 2ms

#─────────────────────────────────
# SSL
#─────────────────────────────────
ssl                      = on
ssl_cert_file            = 'server.crt'
ssl_key_file             = 'server.key'

#─────────────────────────────────
# Locale
#─────────────────────────────────
timezone                         = 'UTC'
default_text_search_config       = 'pg_catalog.english'
```


**Hands-on: Production Config Deploy করো এবং Apply করো**

```bash
# ─── Step 1: Existing config backup নাও ───
sudo cp /var/lib/pgsql/16/data/postgresql.conf \
        /var/lib/pgsql/16/data/postgresql.conf.bak.$(date +%Y%m%d)

# ─── Step 2: Config apply করো (ALTER SYSTEM দিয়ে — safe, reversible) ───
psql -U postgres << 'EOF'
-- Connection
ALTER SYSTEM SET listen_addresses = '*';
ALTER SYSTEM SET max_connections  = 200;

-- Memory (16GB RAM server এর জন্য)
ALTER SYSTEM SET shared_buffers           = '4GB';
ALTER SYSTEM SET work_mem                 = '64MB';
ALTER SYSTEM SET maintenance_work_mem     = '1GB';
ALTER SYSTEM SET effective_cache_size     = '12GB';
ALTER SYSTEM SET wal_buffers              = '64MB';

-- WAL & Checkpoint
ALTER SYSTEM SET wal_level                = 'replica';
ALTER SYSTEM SET synchronous_commit       = 'on';
ALTER SYSTEM SET checkpoint_timeout       = '5min';
ALTER SYSTEM SET max_wal_size             = '2GB';
ALTER SYSTEM SET min_wal_size             = '256MB';
ALTER SYSTEM SET wal_compression          = 'on';

-- Planner (SSD)
ALTER SYSTEM SET random_page_cost         = '1.1';
ALTER SYSTEM SET effective_io_concurrency = '200';

-- Parallel Query
ALTER SYSTEM SET max_parallel_workers_per_gather = 4;
ALTER SYSTEM SET max_parallel_workers            = 8;

-- Logging
ALTER SYSTEM SET logging_collector               = 'on';
ALTER SYSTEM SET log_min_duration_statement      = '1000';
ALTER SYSTEM SET log_checkpoints                 = 'on';
ALTER SYSTEM SET log_lock_waits                  = 'on';
ALTER SYSTEM SET log_autovacuum_min_duration     = '250ms';

-- Autovacuum
ALTER SYSTEM SET autovacuum_max_workers          = 5;
ALTER SYSTEM SET autovacuum_naptime              = '30s';
ALTER SYSTEM SET autovacuum_vacuum_scale_factor  = '0.02';
ALTER SYSTEM SET autovacuum_analyze_scale_factor = '0.01';
ALTER SYSTEM SET autovacuum_vacuum_cost_delay    = '2ms';

-- Idle connection timeout
ALTER SYSTEM SET idle_in_transaction_session_timeout = '10min';
EOF

# ─── Step 3: Restart (shared_buffers restart দরকার) ───
sudo systemctl restart postgresql-16

# ─── Step 4: সব parameter apply হয়েছে verify করো ───
psql -U postgres -c "
SELECT name, setting, unit
FROM pg_settings
WHERE name IN (
    'shared_buffers','work_mem','maintenance_work_mem',
    'effective_cache_size','max_connections','wal_level',
    'random_page_cost','effective_io_concurrency',
    'max_parallel_workers','logging_collector',
    'autovacuum_max_workers','idle_in_transaction_session_timeout'
)
ORDER BY name;"

# ─── Step 5: Pending restart আছে কিনা check করো ───
psql -U postgres -c "
SELECT name, setting, pending_restart
FROM pg_settings
WHERE pending_restart = true;"
# Empty হলে সব apply হয়েছে ✅

# ─── Step 6: postgresql.auto.conf verify করো ───
cat /var/lib/pgsql/16/data/postgresql.auto.conf
```

---

## 2.10 OS Prerequisites — PostgreSQL Install এর আগে

PostgreSQL ভালোভাবে চালাতে OS level কিছু setup করতে হয়।

```bash
# ─────────────────────────────────────────────
# Kernel Parameters (sysctl)
# ─────────────────────────────────────────────
sudo tee /etc/sysctl.d/99-postgresql.conf << 'EOF'
# Shared memory
kernel.shmmax = 17179869184       # 16GB (shared_buffers এর 2× হলে ভালো)
kernel.shmall = 4194304           # 16GB in 4KB pages

# Virtual memory
vm.swappiness = 1                 # Swap কম ব্যবহার করো (PostgreSQL RAM এ থাকবে)
vm.overcommit_memory = 2          # Don't overcommit (OOM killer এড়াতে)
vm.overcommit_ratio = 80

# Dirty page writeback
vm.dirty_background_ratio = 5    # 5% dirty হলে background flush শুরু
vm.dirty_ratio = 30              # 30% dirty হলে application block করে flush

# Huge pages
vm.nr_hugepages = 1024           # 1024 × 2MB = 2GB (shared_buffers 4GB এর জন্য আধা)
# vm.nr_hugepages = shared_buffers_in_MB / 2 + কিছু extra

# Network (high connection এ)
net.core.somaxconn = 4096
net.ipv4.tcp_keepalive_time = 60
net.ipv4.tcp_keepalive_intvl = 10
net.ipv4.tcp_keepalive_probes = 6
EOF
sudo sysctl -p /etc/sysctl.d/99-postgresql.conf

# ─────────────────────────────────────────────
# Transparent Huge Pages — DISABLE (PostgreSQL এ harmful)
# ─────────────────────────────────────────────
sudo tee /etc/systemd/system/disable-thp.service << 'EOF'
[Unit]
Description=Disable Transparent Huge Pages
DefaultDependencies=no
Before=sysinit.target local-fs.target

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/enabled'
ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/defrag'

[Install]
WantedBy=basic.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now disable-thp

# ─────────────────────────────────────────────
# ulimits — postgres user এর resource limits
# ─────────────────────────────────────────────
sudo tee /etc/security/limits.d/99-postgresql.conf << 'EOF'
postgres    soft    nofile    65536
postgres    hard    nofile    65536
postgres    soft    nproc     65536
postgres    hard    nproc     65536
postgres    soft    memlock   unlimited
postgres    hard    memlock   unlimited
EOF

# ─────────────────────────────────────────────
# I/O Scheduler
# ─────────────────────────────────────────────
# SSD: none (kernel 5.x) বা mq-deadline
# HDD: deadline বা cfq
for disk in /sys/block/sd*/queue/scheduler; do
    echo "none" > $disk 2>/dev/null || echo "mq-deadline" > $disk 2>/dev/null
done

# Persistent করো:
sudo tee /etc/udev/rules.d/60-ioschedulers.rules << 'EOF'
ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="none"
ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/rotational}=="1", ATTR{queue/scheduler}="deadline"
EOF

# ─────────────────────────────────────────────
# File System (XFS recommended for PostgreSQL)
# ─────────────────────────────────────────────
# Format:
# sudo mkfs.xfs -f /dev/sdb1
# Mount options:
# /dev/sdb1  /var/lib/pgsql  xfs  noatime,nodiratime,allocsize=16m  0 0
# noatime: access time update করে না → I/O কমে
# allocsize=16m: WAL write এ efficient

# ─────────────────────────────────────────────
# SELinux (Rocky Linux 9)
# ─────────────────────────────────────────────
# Custom data directory তে context দাও
sudo semanage fcontext -a -t postgresql_db_t "/data/postgresql(/.*)?"
sudo restorecon -Rv /data/postgresql/
# অথবা custom port ব্যবহার করলে:
sudo semanage port -a -t postgresql_port_t -p tcp 5433

# ─────────────────────────────────────────────
# Firewall
# ─────────────────────────────────────────────
sudo firewall-cmd --permanent --add-port=5432/tcp
sudo firewall-cmd --reload

# Specific IP allow (more secure):
sudo firewall-cmd --permanent --add-rich-rule='
  rule family=ipv4
  source address=172.16.93.0/24
  port port=5432 protocol=tcp accept'
sudo firewall-cmd --reload
```

---

## 2.11 pg_hba.conf — Authentication Deep Dive

`pg_hba.conf` হলো PostgreSQL এর "দরজার তালা"। Authentication এর সব নিয়ম এখানে।

### File Format

```
# TYPE    DATABASE    USER        ADDRESS             METHOD    OPTIONS
local     all         postgres                        peer
host      mydb        appuser     172.16.93.0/24      scram-sha-256
hostssl   all         all         10.0.0.0/8          scram-sha-256
```

**Rule Processing:** উপর থেকে নিচে, প্রথম match এ stop। Order গুরুত্বপূর্ণ।

### TYPE এর সব Options

```
local     → Unix Domain Socket (same machine, password ছাড়া faster)
host      → TCP/IP (SSL বা non-SSL উভয়)
hostssl   → TCP/IP, SSL required (plain text refuse করে)
hostnossl → TCP/IP, non-SSL only (SSL connection refuse করে)
hostgssenc → GSSAPI encryption (Kerberos)
hostnogssenc → No GSSAPI
```

### DATABASE এর Options

```
all              → সব database
mydb             → specific database নাম
mydb,otherdb     → comma-separated list
@/etc/pgdbs.txt  → file থেকে database list
replication      → replication connections (special keyword)
sameuser         → username = database name (users তে নিজের নামের DB)
samegroup        → user's group = database name
```

### USER এর Options

```
all              → সব user
postgres         → specific user
+myrole          → role এর সব members (+ prefix)
@/etc/pgusers.txt → file থেকে user list
```

### METHOD এর সব Options

| Method | কীভাবে কাজ করে | কখন ব্যবহার করবো |
|---|---|---|
| `trust` | Password নেই, সব allow | কখনো production এ না! |
| `reject` | সবসময় reject | Specific IP block করতে |
| `peer` | OS username = PG username | Unix socket local connections |
| `ident` | TCP ident protocol | Legacy, avoid |
| `md5` | MD5 hash | Legacy, scram এ upgrade করো |
| `scram-sha-256` | SCRAM-SHA-256 | Production standard ✅ |
| `password` | Plain text password | SSL connection এ only |
| `cert` | SSL client certificate | High security, mTLS |
| `ldap` | LDAP/Active Directory | Enterprise SSO |
| `radius` | RADIUS server | Network auth |
| `gss` | Kerberos GSSAPI | Enterprise Kerberos |
| `sspi` | Windows SSPI | Windows AD |
| `pam` | Linux PAM | OS auth integration |

### Production pg_hba.conf Template

```
# TYPE    DATABASE        USER            ADDRESS             METHOD

# ─── Unix Socket (local server) ───
# postgres superuser: peer auth (OS level trust)
local   all             postgres                            peer

# Local users: password required
local   all             all                                 scram-sha-256

# ─── TCP/IP Connections ───
# localhost
host    all             all             127.0.0.1/32        scram-sha-256
host    all             all             ::1/128             scram-sha-256

# Application servers (internal network)
host    mydb            appuser         172.16.93.0/24      scram-sha-256

# DBA access (specific IPs only)
host    all             postgres        172.16.93.10/32     scram-sha-256
host    all             dba_team        172.16.93.0/24      scram-sha-256

# Monitoring user (PMM)
host    all             pmm_monitor     127.0.0.1/32        scram-sha-256

# Replication (standby servers)
host    replication     replicator      172.16.93.141/32    scram-sha-256
host    replication     replicator      172.16.93.142/32    scram-sha-256

# SSL required for sensitive connections
hostssl all             sensitive_user  0.0.0.0/0           scram-sha-256

# Block everything else (implicit, কিন্তু explicit রাখা ভালো)
# host  all             all             0.0.0.0/0           reject
```

### Authentication Options (METHOD এর পরে)

```
# clientcert: SSL client certificate require করো
hostssl all all 0.0.0.0/0 scram-sha-256 clientcert=verify-full

# ldapserver: LDAP server specify করো
host all all 0.0.0.0/0 ldap ldapserver=ldap.company.com ldapprefix="cn=" ldapsuffix=",dc=company,dc=com"

# radiusservers: RADIUS server
host all all 0.0.0.0/0 radius radiusservers="radius.company.com" radiussecret="secret"
```

```bash
# pg_hba.conf reload (restart ছাড়া)
sudo systemctl reload postgresql-16
# অথবা:
psql -c "SELECT pg_reload_conf();"

# Current auth method দেখো
psql -c "SELECT line_number, type, database, user_name, address, auth_method FROM pg_hba_file_rules;"

# Test connection (specific user এর জন্য)
psql -h 172.16.93.140 -U appuser -d mydb -c "SELECT current_user, inet_server_addr();"
```

---

## 2.12 Configuration Tuning Guide — RAM অনুযায়ী

**4GB RAM:**
```ini
shared_buffers           = 1GB
work_mem                 = 16MB
maintenance_work_mem     = 256MB
effective_cache_size     = 3GB
max_connections          = 100
wal_buffers              = 32MB
max_parallel_workers_per_gather = 2
max_parallel_workers     = 4
```

**8GB RAM:**
```ini
shared_buffers           = 2GB
work_mem                 = 32MB
maintenance_work_mem     = 512MB
effective_cache_size     = 6GB
max_connections          = 150
wal_buffers              = 64MB
max_parallel_workers_per_gather = 2
max_parallel_workers     = 4
```

**16GB RAM:**
```ini
shared_buffers           = 4GB
work_mem                 = 64MB
maintenance_work_mem     = 1GB
effective_cache_size     = 12GB
max_connections          = 200
wal_buffers              = 64MB
max_parallel_workers_per_gather = 4
max_parallel_workers     = 8
```

**32GB RAM:**
```ini
shared_buffers           = 8GB
work_mem                 = 128MB
maintenance_work_mem     = 2GB
effective_cache_size     = 24GB
max_connections          = 300
wal_buffers              = 128MB
max_parallel_workers_per_gather = 4
max_parallel_workers     = 8
```

**64GB RAM:**
```ini
shared_buffers           = 16GB
work_mem                 = 256MB
maintenance_work_mem     = 4GB
effective_cache_size     = 48GB
max_connections          = 400       # pgBouncer দিয়ে বাড়াও
wal_buffers              = 256MB
max_parallel_workers_per_gather = 8
max_parallel_workers     = 16
```

**Workload-specific tuning:**
```ini
# OLTP (many short transactions, high concurrency):
work_mem                 = 4-16MB  # Conservative (many concurrent)
max_connections          = 200+    # pgBouncer strongly recommended
checkpoint_timeout       = 5min    # Frequent checkpoint
synchronous_commit       = on      # Durability priority

# Analytics/OLAP (few long queries, heavy aggregates):
work_mem                 = 256MB-1GB  # Large (few concurrent)
max_parallel_workers_per_gather = 8   # Use all cores
max_connections          = 20-50      # Few concurrent users
checkpoint_timeout       = 15min      # Less frequent
synchronous_commit       = off        # Speed priority (if acceptable)
jit                      = on         # JIT helps complex queries

# Mixed (web app + reporting):
work_mem                 = 64MB       # Balance
max_connections          = 200        # pgBouncer
# Analytics sessions:
# SET work_mem = '512MB';  -- per session
```

**Hands-on: RAM Size Detect করো এবং Recommended Config Apply করো**

```bash
# Step 1: তোমার server এর RAM দেখো
free -h
# অথবা:
grep MemTotal /proc/meminfo

TOTAL_RAM_MB=$(grep MemTotal /proc/meminfo | awk '{print int($2/1024)}')
echo "Total RAM: ${TOTAL_RAM_MB}MB"

# Step 2: Recommended values calculate করো
SHARED_BUFFERS=$((TOTAL_RAM_MB / 4))
EFFECTIVE_CACHE=$((TOTAL_RAM_MB * 3 / 4))
MAINT_WORK_MEM=$((TOTAL_RAM_MB / 16))
WAL_BUFFERS=64

echo "--- Recommended Settings ---"
echo "shared_buffers           = ${SHARED_BUFFERS}MB"
echo "effective_cache_size     = ${EFFECTIVE_CACHE}MB"
echo "maintenance_work_mem     = ${MAINT_WORK_MEM}MB"
echo "wal_buffers              = ${WAL_BUFFERS}MB"

# Step 3: Apply করো
psql -U postgres << EOF
ALTER SYSTEM SET shared_buffers = '${SHARED_BUFFERS}MB';
ALTER SYSTEM SET effective_cache_size = '${EFFECTIVE_CACHE}MB';
ALTER SYSTEM SET maintenance_work_mem = '${MAINT_WORK_MEM}MB';
ALTER SYSTEM SET wal_buffers = '${WAL_BUFFERS}MB';
EOF

# Step 4: Restart (shared_buffers restart দরকার)
sudo systemctl restart postgresql-16

# Step 5: Verify
psql -U postgres -c "SHOW shared_buffers; SHOW effective_cache_size; SHOW maintenance_work_mem;"
```

```bash
# ─── PGTune — automatic configuration generator ───
# Online tool: https://pgtune.leopard.in.ua
# CLI দিয়েও করা যায়:

DB_VERSION=16
TOTAL_RAM="16GB"
CPU_COUNT=$(nproc)
CONNECTIONS=200
WORKLOAD="web"   # web / oltp / dw / desktop / mixed

# PGTune এর logic manually calculate করো:
python3 << 'PYEOF'
import math
ram_gb = 16
cpu = 4
conn = 200

shared_buffers     = int(ram_gb * 0.25 * 1024)  # MB
work_mem           = int((ram_gb * 1024 * 0.25) / (conn * 3))  # MB
maint_work_mem     = min(int(ram_gb * 1024 / 16), 2048)  # MB
effective_cache    = int(ram_gb * 0.75 * 1024)  # MB
wal_buffers        = min(int(shared_buffers * 0.03), 64)  # MB

print(f"shared_buffers           = {shared_buffers}MB")
print(f"work_mem                 = {work_mem}MB")
print(f"maintenance_work_mem     = {maint_work_mem}MB")
print(f"effective_cache_size     = {effective_cache}MB")
print(f"wal_buffers              = {wal_buffers}MB")
print(f"max_parallel_workers_per_gather = {max(2, cpu//2)}")
PYEOF
```

---

## 2.13 Parameter Tuning Workflow

```
Step 1: Baseline নাও
  pgBench দিয়ে current TPS/latency measure করো
  pg_stat_statements reset করো
  24 ঘন্টা চালাও

Step 2: Bottleneck identify করো
  Buffer hit rate < 99% → shared_buffers বাড়াও
  Temp files বেশি → work_mem বাড়াও
  Checkpoint too frequent → max_wal_size বাড়াও
  Lock waits → query/application optimize করো
  High CPU → parallel workers tune, JIT

Step 3: একটা Parameter change করো
  একসাথে অনেক change করলে কোনটায় improvement হলো বোঝা যায় না

Step 4: Restart/Reload করো
  Restart: postmaster context parameters (shared_buffers)
  Reload: sighup context parameters (work_mem)

Step 5: Measure করো (24 ঘন্টা)
  আগের baseline এর সাথে compare করো

Step 6: Document করো
  কোন change কতটা improvement এনেছে
```

```sql
-- Parameter change history দেখো
SELECT name, setting, unit, source, sourcefile, sourceline
FROM pg_settings
WHERE source NOT IN ('default', 'environment variable')
ORDER BY name;
-- source = 'configuration file' → postgresql.conf
-- source = 'override' → ALTER SYSTEM

-- Pending restart check
SELECT name, setting, pending_restart
FROM pg_settings
WHERE pending_restart = true;

-- Parameter এর default থেকে কতটা আলাদা
SELECT name, setting, boot_val AS default_value, reset_val
FROM pg_settings
WHERE setting != boot_val
  AND context NOT IN ('internal')
ORDER BY name;
```

**Hands-on: Complete Tuning Workflow চালাও**

```bash
# ─── Step 1: Baseline measure করো (pgBench দিয়ে) ───
createdb -U postgres pgbench_test
pgbench -U postgres -i -s 10 pgbench_test   # Initialize test data
pgbench -U postgres -c 10 -j 2 -T 60 pgbench_test 2>&1 | tee /tmp/baseline.txt
grep "tps\|latency" /tmp/baseline.txt
# Note করো: tps, avg latency

# ─── Step 2: Current bottleneck identify করো ───
psql -U postgres << 'EOF'
-- Buffer hit rate (কম হলে shared_buffers বাড়াও)
SELECT 'buffer_hit_rate',
    ROUND(sum(heap_blks_hit)*100.0/NULLIF(sum(heap_blks_hit)+sum(heap_blks_read),0), 2)
FROM pg_statio_user_tables;

-- Temp file usage (বেশি হলে work_mem বাড়াও)
SELECT 'temp_files', temp_files, pg_size_pretty(temp_bytes)
FROM pg_stat_database WHERE datname = 'pgbench_test';

-- Checkpoint frequency (বেশি forced হলে max_wal_size বাড়াও)
SELECT 'checkpoint_ratio',
    checkpoints_req, checkpoints_timed,
    ROUND(checkpoints_req*100.0/NULLIF(checkpoints_req+checkpoints_timed,0),1) AS forced_pct
FROM pg_stat_bgwriter;
EOF

# ─── Step 3: Parameter change করো (একটা একটা করে) ───
# Example: work_mem বাড়াও
psql -U postgres -c "ALTER SYSTEM SET work_mem = '64MB';"
psql -U postgres -c "SELECT pg_reload_conf();"   # reload (restart নয়)

# ─── Step 4: After-change measure করো ───
pgbench -U postgres -c 10 -j 2 -T 60 pgbench_test 2>&1 | tee /tmp/after_tuning.txt
grep "tps\|latency" /tmp/after_tuning.txt

# ─── Step 5: Compare করো ───
echo "=== BEFORE ===" && grep tps /tmp/baseline.txt
echo "=== AFTER  ===" && grep tps /tmp/after_tuning.txt

# ─── Step 6: Statistics reset করো (fresh measurement এর জন্য) ───
psql -U postgres -c "SELECT pg_stat_reset();"
psql -U postgres -c "SELECT pg_stat_statements_reset();" 2>/dev/null

# Cleanup
dropdb -U postgres pgbench_test
```

```bash
# ─── Production Tuning Checklist Script ───
cat > /usr/local/bin/pg_tune_check.sh << 'SCRIPT'
#!/bin/bash
PSQL="psql -U postgres -t -A -c"

echo "=== PostgreSQL Tuning Opportunities ==="

# 1. Buffer hit rate
HIT=$(${PSQL} "SELECT ROUND(sum(heap_blks_hit)*100.0/NULLIF(sum(heap_blks_hit)+sum(heap_blks_read),0),1) FROM pg_statio_user_tables;")
echo ""
echo "Buffer Hit Rate: ${HIT}%"
if (( $(echo "$HIT < 99" | bc -l) )); then
    CURRENT=$(${PSQL} "SHOW shared_buffers;")
    echo "  → shared_buffers বাড়াও (current: $CURRENT)"
fi

# 2. Temp files
TMPFILES=$(${PSQL} "SELECT sum(temp_files) FROM pg_stat_database;")
TMPSIZE=$(${PSQL} "SELECT pg_size_pretty(sum(temp_bytes)) FROM pg_stat_database;")
echo ""
echo "Temp Files Created: ${TMPFILES} (${TMPSIZE})"
if [ "$TMPFILES" -gt 0 ]; then
    CURRENT=$(${PSQL} "SHOW work_mem;")
    echo "  → work_mem বাড়াও (current: $CURRENT)"
fi

# 3. Checkpoint frequency
FORCED=$(${PSQL} "SELECT checkpoints_req FROM pg_stat_bgwriter;")
TIMED=$(${PSQL} "SELECT checkpoints_timed FROM pg_stat_bgwriter;")
TOTAL=$((FORCED + TIMED))
if [ "$TOTAL" -gt 0 ]; then
    PCT=$((FORCED * 100 / TOTAL))
    echo ""
    echo "Forced Checkpoints: ${PCT}% (${FORCED} of ${TOTAL})"
    if [ "$PCT" -gt 20 ]; then
        CURRENT=$(${PSQL} "SHOW max_wal_size;")
        echo "  → max_wal_size বাড়াও (current: $CURRENT)"
    fi
fi

# 4. Connections
CONN=$(${PSQL} "SELECT count(*) FROM pg_stat_activity;")
MAX=$(${PSQL} "SELECT setting FROM pg_settings WHERE name='max_connections';")
PCT=$((CONN * 100 / MAX))
echo ""
echo "Connection Utilization: ${CONN}/${MAX} (${PCT}%)"
if [ "$PCT" -gt 70 ]; then
    echo "  → pgBouncer install করো বা max_connections বাড়াও"
fi

echo ""
echo "=== Done ==="
SCRIPT
chmod +x /usr/local/bin/pg_tune_check.sh
sudo -u postgres /usr/local/bin/pg_tune_check.sh
```

---

# 3. Data Types

## 3.1 Numeric Types

### Integer

```
┌──────────────┬──────────┬─────────────────────────────────────┐
│ Type         │ Storage  │ Range                               │
├──────────────┼──────────┼─────────────────────────────────────┤
│ SMALLINT     │ 2 bytes  │ -32,768 to 32,767                   │
│ INTEGER      │ 4 bytes  │ -2,147,483,648 to 2,147,483,647     │
│ BIGINT       │ 8 bytes  │ ±9.2 quintillion                    │
└──────────────┴──────────┴─────────────────────────────────────┘
```

**MySQL vs PostgreSQL:**
```
MySQL:      TINYINT(1B), SMALLINT(2B), MEDIUMINT(3B), INT(4B), BIGINT(8B)
            UNSIGNED keyword আছে (0 to positive range doubles)

PostgreSQL: SMALLINT, INTEGER, BIGINT (TINYINT নেই, UNSIGNED নেই)
            Positive-only চাইলে CHECK constraint ব্যবহার করো:
            age SMALLINT CHECK (age >= 0)
```

**Auto Increment — PostgreSQL 10+ IDENTITY preferred:**

```sql
-- ❌ পুরনো way (SERIAL) — এখনো কাজ করে কিন্তু deprecated
id SERIAL PRIMARY KEY           -- SEQUENCE implicitly তৈরি করে

-- ✅ নতুন way (IDENTITY) — SQL standard, recommended
id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY
   -- ALWAYS = manually insert করা যাবে না
id BIGINT  GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY
   -- BY DEFAULT = manually insert করা যাবে (import, restore এ দরকার)

-- Sequence manually control করতে চাইলে
CREATE SEQUENCE users_id_seq START 1000;
id INTEGER DEFAULT nextval('users_id_seq') PRIMARY KEY
```

### Decimal/Numeric

```sql
-- ❌ টাকার জন্য FLOAT বা DOUBLE ব্যবহার করো না!
SELECT 0.1 + 0.2;  -- Result: 0.30000000000000004  ← floating point error!

-- ✅ NUMERIC বা DECIMAL (এরা same) ব্যবহার করো
price  NUMERIC(10, 2)   -- Total 10 digits, 2 after decimal (max: 99999999.99)
salary NUMERIC(12, 2)
amount NUMERIC(15, 4)   -- High precision চাইলে

-- Scientific calculation এ FLOAT/DOUBLE okay
-- কিন্তু money, quantity — সবসময় NUMERIC
```

---

## 3.2 Character Types

```sql
CHAR(n)   -- Fixed length, n bytes, trailing spaces দিয়ে pad করে
          -- শুধু exactly n-character data এর জন্য (country code, status code)

VARCHAR(n) -- Variable length, maximum n characters
           -- MySQL এর মতো

TEXT       -- Unlimited length
           -- PostgreSQL এ TEXT অনেক efficient
           -- VARCHAR(n) এর চেয়ে TEXT + CHECK constraint better
```

**PostgreSQL এ TEXT সবচেয়ে ভালো:**
```sql
-- MySQL তে TEXT এ সমস্যা আছে (no default, limited index)
-- PostgreSQL এ TEXT তে কোনো সমস্যা নেই

-- ✅ PostgreSQL recommended approach
name  TEXT NOT NULL CHECK (char_length(name) <= 200)
email TEXT NOT NULL CHECK (email ~* '^[^@]+@[^@]+\.[^@]+$')

-- VARCHAR(200) এর চেয়ে TEXT + CHECK পছন্দনীয়
-- Performance identical — CHECK constraint compile হয় rule এ
```

---

## 3.3 Date and Time Types

```
DATE         → 4 bytes  → '2024-03-08'
TIME         → 8 bytes  → '14:30:00'
TIMETZ       → 12 bytes → '14:30:00+06:00' (time with timezone)
TIMESTAMP    → 8 bytes  → '2024-03-08 14:30:00' (no timezone stored)
TIMESTAMPTZ  → 8 bytes  → '2024-03-08 08:30:00 UTC' (UTC এ stored)
INTERVAL     → 16 bytes → '1 year 2 months 3 days 4 hours'
```

**MySQL vs PostgreSQL:**
```
MySQL DATETIME   ≈ PostgreSQL TIMESTAMP    (no timezone)
MySQL TIMESTAMP  ≈ PostgreSQL TIMESTAMPTZ  (timezone-aware, UTC stored)

PostgreSQL এর TIMESTAMPTZ:
  Storage: সবসময় UTC তে store
  Display: Client এর timezone অনুযায়ী show
  SET timezone = 'Asia/Dhaka';
  → UTC time + 6h = Dhaka time দেখাবে
```

```sql
-- Best practice — সবসময় TIMESTAMPTZ ব্যবহার করো
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()

-- updated_at auto-update করতে trigger দরকার (MySQL এর ON UPDATE CURRENT_TIMESTAMP নেই)
CREATE OR REPLACE FUNCTION fn_update_timestamp()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_users_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION fn_update_timestamp();

-- INTERVAL ব্যবহার
SELECT NOW() - INTERVAL '7 days';           -- 7 দিন আগে
SELECT created_at + INTERVAL '1 month'      -- 1 মাস পরে
FROM subscriptions;
SELECT AGE(NOW(), created_at) AS account_age FROM users; -- কতদিন হলো
```

---

## 3.4 Boolean

```sql
is_active   BOOLEAN NOT NULL DEFAULT TRUE
is_verified BOOLEAN NOT NULL DEFAULT FALSE

-- Accepted values:
-- TRUE:  true, 't', 'yes', 'on', '1', TRUE
-- FALSE: false, 'f', 'no', 'off', '0', FALSE

-- MySQL এ BOOLEAN = TINYINT(1) (0/1)
-- PostgreSQL এ আলাদা data type, 1 byte

-- Query
SELECT * FROM users WHERE is_active;           -- WHERE is_active = TRUE
SELECT * FROM users WHERE NOT is_verified;     -- WHERE is_verified = FALSE
```

---

## 3.5 JSON vs JSONB — কোনটা কখন

```
JSON:
  Text হিসেবে store হয়
  Insert order preserve হয়
  Duplicate keys রাখে
  Read এ parse করে → slow queries
  শুধু exact JSON store করতে চাইলে (log, audit trail)

JSONB:
  Binary format এ store হয়
  Write এর সময় parse হয় → write একটু slow
  Read এ parse করতে হয় না → fast queries
  GIN index দেওয়া যায়
  Duplicate keys রাখে না
  → সাধারণত JSONB ব্যবহার করো
```

```sql
CREATE TABLE products (
    id       BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name     TEXT NOT NULL,
    metadata JSONB     -- JSON নয়, JSONB
);

-- Insert
INSERT INTO products (name, metadata)
VALUES ('Laptop', '{"brand": "Dell", "specs": {"ram": 16, "ssd": 512}, "tags": ["gaming", "business"]}');

-- Operators:
-- ->   JSON object field বের করো (JSON type return করে)
-- ->>  JSON object field বের করো (TEXT type return করে)
-- #>   Nested path (JSON return)
-- #>> Nested path (TEXT return)
-- @>   Left এ right contain আছে কি?
-- ?    Key exist করে কি?

SELECT metadata -> 'brand'         FROM products;  -- "Dell" (JSON)
SELECT metadata ->> 'brand'        FROM products;  -- Dell  (TEXT)
SELECT metadata #>> '{specs,ram}'  FROM products;  -- 16    (TEXT)
SELECT * FROM products WHERE metadata @> '{"brand": "Dell"}';  -- contains
SELECT * FROM products WHERE metadata ? 'brand';   -- key exists

-- GIN Index (JSONB এ)
CREATE INDEX idx_metadata ON products USING GIN(metadata);
-- এখন @>, ?, ?& operators এ index use হবে

-- Specific key এ index
CREATE INDEX idx_brand ON products ((metadata ->> 'brand'));

-- Generated column দিয়ে
ALTER TABLE products
    ADD COLUMN brand TEXT GENERATED ALWAYS AS (metadata ->> 'brand') STORED;
CREATE INDEX idx_brand ON products(brand);
SELECT * FROM products WHERE brand = 'Dell';
```

---

## 3.6 Arrays — Column এ List Store

MySQL তে নেই। PostgreSQL unique feature।

```sql
CREATE TABLE articles (
    id      BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    title   TEXT NOT NULL,
    tags    TEXT[],          -- Text array
    scores  INTEGER[],       -- Integer array
    matrix  INTEGER[][]      -- 2D array
);

INSERT INTO articles (title, tags)
VALUES ('PostgreSQL Guide', ARRAY['database', 'sql', 'tutorial']);
-- অথবা: '{"database", "sql", "tutorial"}'

-- Query
SELECT * FROM articles WHERE 'database' = ANY(tags);      -- element search
SELECT * FROM articles WHERE tags @> ARRAY['database'];   -- contains
SELECT * FROM articles WHERE tags && ARRAY['sql', 'nosql']; -- overlap

-- Array functions
SELECT array_length(tags, 1) FROM articles;    -- length
SELECT unnest(tags) AS tag FROM articles;       -- array → multiple rows
SELECT array_agg(tag) FROM (SELECT unnest(tags) AS tag FROM articles) t; -- rows → array

-- GIN Index
CREATE INDEX idx_tags ON articles USING GIN(tags);
SELECT * FROM articles WHERE tags @> ARRAY['database'];  -- index use করবে
```

---

## 3.7 UUID

```sql
-- PostgreSQL 13+ এ built-in (extension লাগে না)
id UUID DEFAULT gen_random_uuid() PRIMARY KEY

-- পুরনো version এ extension লাগতো
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
id UUID DEFAULT uuid_generate_v4() PRIMARY KEY

-- UUID vs BIGINT as PK:
-- UUID: Globally unique (distributed system এ useful), random → index fragmentation
-- BIGINT: Sequential → index efficient, smaller (8B vs 16B)
-- Production: BIGINT IDENTITY prefer করো যদি না distributed system হয়
```

---

## 3.8 Range Types — Period এবং Range Data

```sql
-- Built-in range types
DATERANGE      -- [2024-01-01, 2024-12-31)
TSRANGE        -- TIMESTAMP range
TSTZRANGE      -- TIMESTAMPTZ range
INT4RANGE      -- INTEGER range
INT8RANGE      -- BIGINT range
NUMRANGE       -- NUMERIC range

-- Real use case: Hotel booking — overlapping booking prevent
CREATE TABLE bookings (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    room_id     INTEGER NOT NULL,
    guest_name  TEXT NOT NULL,
    stay_period DATERANGE NOT NULL,

    -- Constraint: same room এ overlapping booking allow নয়
    EXCLUDE USING GIST (room_id WITH =, stay_period WITH &&)
);

INSERT INTO bookings (room_id, guest_name, stay_period)
VALUES (101, 'Alice', '[2024-03-01, 2024-03-07)');
-- '[' = inclusive, ')' = exclusive (check-out day এ পরের guest আসতে পারে)

-- Overlap check query
SELECT * FROM bookings
WHERE room_id = 101
  AND stay_period && '[2024-03-04, 2024-03-10)';

-- GiST Index
CREATE INDEX idx_stay ON bookings USING GIST(stay_period);

-- এই ধরনের constraint MySQL এ application level এ করতে হয়
-- PostgreSQL এ database level এ guarantee
```

---

## 3.9 Practical Table Design

```sql
CREATE TABLE users (
    -- Primary Key: BIGINT IDENTITY (UUID নয়, sequential ভালো)
    id           BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,

    -- Unique identifiers
    username     TEXT NOT NULL UNIQUE
                     CHECK (char_length(username) BETWEEN 3 AND 50),
    email        TEXT NOT NULL UNIQUE
                     CHECK (email ~* '^[^@]+@[^@]+\.[^@]+$'),

    -- Basic info
    full_name    TEXT NOT NULL
                     CHECK (char_length(full_name) BETWEEN 2 AND 150),
    bio          TEXT,       -- TEXT, no length limit

    -- Fixed-length codes
    country_code CHAR(2) NOT NULL DEFAULT 'BD',

    -- Enum-like: TEXT + CHECK (PostgreSQL এ ENUM type আছে কিন্তু TEXT + CHECK flexible)
    status       TEXT NOT NULL DEFAULT 'active'
                     CHECK (status IN ('active', 'inactive', 'banned')),

    -- Numbers
    age          SMALLINT CHECK (age BETWEEN 0 AND 150),
    login_count  INTEGER NOT NULL DEFAULT 0,
    balance      NUMERIC(12, 2) NOT NULL DEFAULT 0.00,

    -- Dates
    date_of_birth DATE,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_login_at TIMESTAMPTZ,

    -- Boolean
    is_verified   BOOLEAN NOT NULL DEFAULT FALSE,
    is_admin      BOOLEAN NOT NULL DEFAULT FALSE,

    -- Flexible data
    metadata      JSONB,
    tags          TEXT[]
);

-- updated_at auto-trigger
CREATE TRIGGER trg_users_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION fn_update_timestamp();

-- Indexes
CREATE INDEX idx_users_email       ON users(email);
CREATE INDEX idx_users_status      ON users(status) WHERE status != 'active';  -- partial
CREATE INDEX idx_users_country     ON users(country_code);
CREATE INDEX idx_users_created     ON users(created_at DESC);
CREATE INDEX idx_users_metadata    ON users USING GIN(metadata);
CREATE INDEX idx_users_tags        ON users USING GIN(tags);
```

---

# 4. Indexes

## 4.1 Index কেন দরকার

```
Index ছাড়া (Sequential Scan):
  বইয়ের প্রথম পাতা থেকে শেষ পর্যন্ত পড়া
  1 মিলিয়ন row থেকে 1 row খুঁজতে 1 মিলিয়ন comparison
  O(n)

Index দিয়ে (Index Scan):
  বইয়ের পেছনের Index section → সরাসরি page number
  1 মিলিয়ন row থেকে 1 row খুঁজতে ~20 comparison (B-Tree height)
  O(log n)
```

PostgreSQL এ **৬ ধরনের** index আছে — MySQL এ শুধু B-Tree এবং Full-Text। এটা PostgreSQL এর বড় সুবিধা।

---

## 4.2 B-Tree Index — Default এবং সবচেয়ে Common

```sql
-- Default index type (USING BTREE লেখা না হলেও B-Tree)
CREATE INDEX idx_email ON users(email);
CREATE UNIQUE INDEX idx_username ON users(username);

-- Composite index
CREATE INDEX idx_country_status ON users(country_code, status);
```

**B-Tree structure:**
```
              [50]
             /    \
       [25]          [75]
      /    \        /    \
  [10,20] [30,40] [60,70] [80,90]

Sorted order → equality (=), range (>, <, BETWEEN), sort (ORDER BY) সব কাজ করে
```

**Left-Prefix Rule — Composite Index এর সবচেয়ে গুরুত্বপূর্ণ নিয়ম:**
```sql
CREATE INDEX idx_a_b_c ON orders(country, status, created_at);

-- ✅ Index USE হবে:
WHERE country = 'BD'
WHERE country = 'BD' AND status = 'active'
WHERE country = 'BD' AND status = 'active' AND created_at > '2024-01-01'
WHERE country = 'BD' ORDER BY status       -- sort এও
WHERE country = 'BD' AND status = 'active' ORDER BY created_at

-- ❌ Index USE হবে না:
WHERE status = 'active'                    -- leftmost (country) নেই
WHERE created_at > '2024-01-01'           -- leftmost নেই
WHERE status = 'active' AND created_at > '2024-01-01'  -- leftmost নেই

-- কারণ: B-Tree এ first column দিয়ে sort হয়
-- country ছাড়া শুরু করলে আমরা tree তে কোথায় আছি জানি না
```

**Composite Index Design Rule:**
```
Equality conditions আগে → Range conditions পরে → Sort column শেষে

Query: WHERE country = 'BD' AND status = 'active' AND created_at > '2024-01-01' ORDER BY created_at
Index: (country, status, created_at)
        equality  equality  range+sort
```

---

## 4.3 Hash Index — Equality Only, Faster

```sql
CREATE INDEX idx_id_hash ON users USING HASH(id);

-- শুধু equality (=) এ কাজ করে
SELECT * FROM users WHERE id = 5;    -- Hash index use করবে

-- Range এ কাজ করে না:
SELECT * FROM users WHERE id > 5;    -- Seq scan হবে

-- PostgreSQL 10+ এ WAL-logged (safe to use, crash recovery supported)
-- B-Tree এর চেয়ে equality তে faster কারণ O(1) lookup
-- কিন্তু range, sort এ কাজ নেই — specific use case এ ব্যবহার করো
```

---

## 4.4 GIN Index — Full-Text, Array, JSONB

**GIN (Generalized Inverted Index)** — ভেতরে থাকা elements এ search করার জন্য।

```
কখন ব্যবহার করবো:
  - JSONB column এ key/value search
  - Array column এ element search
  - Full-text search
  - pg_trgm (partial text search)
```

```sql
-- JSONB এ
CREATE INDEX idx_metadata ON products USING GIN(metadata);
SELECT * FROM products WHERE metadata @> '{"brand": "Dell"}';
SELECT * FROM products WHERE metadata ? 'discount';

-- Array এ
CREATE INDEX idx_tags ON articles USING GIN(tags);
SELECT * FROM articles WHERE tags @> ARRAY['postgresql'];
SELECT * FROM articles WHERE tags && ARRAY['sql', 'database'];

-- Full-text search (tsvector)
ALTER TABLE articles
    ADD COLUMN search_vector TSVECTOR
    GENERATED ALWAYS AS (to_tsvector('english', title || ' ' || COALESCE(body, ''))) STORED;
CREATE INDEX idx_fts ON articles USING GIN(search_vector);
SELECT * FROM articles WHERE search_vector @@ to_tsquery('english', 'postgresql & index');

-- pg_trgm — partial text search (LIKE '%keyword%')
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_trgm ON products USING GIN(name gin_trgm_ops);
SELECT * FROM products WHERE name LIKE '%phone%';   -- এখন index use করবে!
SELECT * FROM products WHERE name ILIKE '%PHONE%';  -- case-insensitive ও
```

---

## 4.5 GiST Index — Geometric, Range, Full-Text

```sql
-- Range type এ (overlap query)
CREATE INDEX idx_booking_period ON bookings USING GIST(stay_period);
SELECT * FROM bookings WHERE stay_period && '[2024-03-04, 2024-03-10)';
-- EXCLUDE USING GIST constraint এও ব্যবহার হয়

-- PostGIS geographic data এ
-- CREATE INDEX idx_location ON places USING GIST(geom);

-- Full-text search (GIN এর alternative)
-- GIN: Insert slow, Query fast
-- GiST: Insert fast, Query relatively slow
-- → Full-text এ GIN prefer করো
```

---

## 4.6 BRIN Index — Large Sequential Table এ

**BRIN (Block Range Index)** — table এর physical order এর সাথে data naturally sorted থাকলে কাজ করে।

```sql
-- Time-series, log, audit table এ
CREATE INDEX idx_events_created ON events USING BRIN(created_at);

-- কেন BRIN?
-- B-Tree index size: table এর ~3-5%
-- BRIN index size:   table এর ~0.001%!
-- BRIN: "Block 1-128 তে created_at: 2024-01-01 to 2024-01-05 আছে"
-- Query: 2024-01-03 এর data → শুধু সেই blocks দেখো

-- কখন কাজ করে না?
-- Random data (UUID primary key) → BRIN ineffective
-- Frequently updated column → physical order নষ্ট হয়
```

---

## 4.7 Partial Index — Subset of Rows

Full table এ index না দিয়ে শুধু relevant rows এ দাও।

```sql
-- শুধু active user এর email এ index
-- inactive, banned user এর email query rare → তাদের index দরকার নেই
CREATE INDEX idx_active_email ON users(email)
    WHERE status = 'active';

-- শুধু pending orders এ index
CREATE INDEX idx_pending_orders ON orders(created_at)
    WHERE status = 'pending';

-- NULL নয় এমন rows এ
CREATE INDEX idx_verified_users ON users(email)
    WHERE last_login_at IS NOT NULL;

-- কেন Partial Index ভালো?
-- Index size ছোট → memory তে বেশি fit করে
-- Write overhead কম → শুধু qualifying rows এ update
-- Query performance ভালো → ছোট index scan
```

---

## 4.8 Expression Index — Function এর Result এ Index

```sql
-- Case-insensitive email search
CREATE INDEX idx_lower_email ON users(LOWER(email));
SELECT * FROM users WHERE LOWER(email) = 'alice@example.com';
-- → Index use করবে (MySQL এ এটা সম্ভব না, Function on column index নষ্ট করে)

-- Date part এ
CREATE INDEX idx_order_year ON orders(EXTRACT(YEAR FROM created_at));
SELECT * FROM orders WHERE EXTRACT(YEAR FROM created_at) = 2024;

-- Computed value এ
CREATE INDEX idx_full_name ON users(first_name || ' ' || last_name);
SELECT * FROM users WHERE first_name || ' ' || last_name = 'Alice Smith';

-- JSON field এ
CREATE INDEX idx_brand ON products((metadata ->> 'brand'));
SELECT * FROM products WHERE metadata ->> 'brand' = 'Dell';
```

---

## 4.9 Covering Index — Index Only Scan

INCLUDE clause দিয়ে extra columns index এ রাখো।

```sql
-- Query: শুধু status এবং total দরকার, user_id দিয়ে filter
SELECT status, total FROM orders WHERE user_id = 5;

-- Index দাও covering এ:
CREATE INDEX idx_user_covering ON orders(user_id) INCLUDE (status, total);
-- user_id = index key (filter এ)
-- status, total = included columns (heap access লাগবে না)

-- EXPLAIN এ দেখবে: Index Only Scan ← heap কে touch করেনি!
-- Visibility Map clean থাকলে heap access লাগে না
-- → সবচেয়ে fast scan type
```

---

## 4.10 EXPLAIN — Execution Plan Analysis

**EXPLAIN** বলে দেয় PostgreSQL কীভাবে query চালাবে। Slow query optimize করার সবচেয়ে গুরুত্বপূর্ণ tool।

```sql
-- Basic plan (query execute করে না)
EXPLAIN SELECT * FROM users WHERE email = 'test@example.com';

-- Actual execution + buffer info (query execute করে)
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM users WHERE email = 'test@example.com';

-- সব details
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT TEXT) SELECT ...;
```

**Output পড়ার নিয়ম — নিচ থেকে উপরে:**
```
EXPLAIN (ANALYZE, BUFFERS) SELECT u.name, count(o.id)
FROM users u JOIN orders o ON u.id = o.user_id
WHERE u.country = 'BD' GROUP BY u.name;

Finalize GroupAggregate  (cost=1234..1300 rows=500 width=40)
                          actual time=15.2..16.1 rows=487 loops=1
  -> Sort  (cost=1234..1247 rows=500)
       actual time=14.9..15.0 rows=500 loops=1
     -> Hash Join  (cost=150..1200 rows=500)
          actual time=2.1..14.2 rows=500 loops=1
          Hash Cond: (o.user_id = u.id)
          Buffers: hit=1250 read=30     ← 1250 pages cache তে, 30 disk read
          -> Seq Scan on orders  (cost=0..800)  ← orders full scan
                actual rows=1000000 loops=1
          -> Hash  (cost=120..120)
               -> Index Scan on users  (cost=0.29..120)
                    Index Cond: (country = 'BD')
                    actual rows=487 loops=1

নিচ থেকে পড়ো:
  1. users table: Index Scan on country → 487 rows (fast)
  2. Hash তৈরি করো users এর result দিয়ে
  3. orders: Seq Scan → 1 million rows! (slow — index দরকার)
  4. Hash Join করো
  5. Sort করো
  6. Aggregate করো
```

**Scan Types — গুরুত্বের ক্রম:**
| Scan Type | মানে | কখন দেখলে |
|---|---|---|
| Index Only Scan | Index মাত্র, heap access নেই | সবচেয়ে ভালো ✅ |
| Index Scan | Index + heap access | ভালো ✅ |
| Bitmap Index Scan | Multiple index → bitmap → heap | OK ✅ |
| Bitmap Heap Scan | Bitmap দিয়ে heap read | OK ✅ |
| Seq Scan | Full table scan | Large table এ দেখলে → index দাও ⚠️ |

**Key fields:**
```
cost=0.00..8.30     → startup cost .. total cost (arbitrary units)
rows=1              → Estimated rows
width=40            → Estimated bytes per row
actual time=0.1..0.2 → Real execution time (ANALYZE সহ)
actual rows=1       → Real rows (ANALYZE সহ)
loops=1             → কতবার চলেছে (nested loop join এ বাড়ে)
Buffers: hit=5 read=2 → Shared Buffer hit / disk read
```

---

## 4.11 Index Design Rules

```sql
-- Rule 1: WHERE clause column এ index দাও
CREATE INDEX idx_user_id ON orders(user_id);
CREATE INDEX idx_status  ON orders(status);

-- Rule 2: JOIN column এ index দাও
CREATE INDEX idx_order_items_order_id ON order_items(order_id);

-- Rule 3: ORDER BY column এ index দাও (filesort এড়াতে)
CREATE INDEX idx_created_desc ON orders(created_at DESC);

-- Rule 4: Frequently filtered + low cardinality → Partial Index
CREATE INDEX idx_pending ON orders(created_at) WHERE status = 'pending';

-- Rule 5: Function on column → Expression Index
CREATE INDEX idx_lower_email ON users(LOWER(email));

-- Rule 6: Covering Index for Index Only Scan
CREATE INDEX idx_user_status ON orders(user_id) INCLUDE (status, total);

-- Rule 7: JSONB/Array → GIN Index
CREATE INDEX idx_tags ON articles USING GIN(tags);

-- Rule 8: Sequential data (time-series) → BRIN
CREATE INDEX idx_events ON events USING BRIN(occurred_at);

-- ❌ করো না: বেশি index দিও না
-- প্রতিটা INSERT/UPDATE/DELETE সব index update করে → write slow
-- Table এ 5-6 এর বেশি index → carefully consider করো

-- Unused index খোঁজো এবং drop করো
SELECT schemaname, tablename, indexname,
    idx_scan AS times_used,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexname NOT LIKE '%pkey%'
ORDER BY pg_relation_size(indexrelid) DESC;
-- idx_scan = 0 মানে server start থেকে কখনো use হয়নি → drop candidate
```

---

# 5. Transactions & Locking

## 5.1 Transaction কী এবং কেন

Transaction মানে কয়েকটা operation কে একটা unit হিসেবে treat করা। হয় সবগুলো হবে, নয়তো কিছুই না।

```sql
-- ব্যাংক transfer — দুটো operation এক unit এ হতে হবে
BEGIN;
    UPDATE accounts SET balance = balance - 5000 WHERE user_id = 1;
    UPDATE accounts SET balance = balance + 5000 WHERE user_id = 2;
COMMIT;
-- মাঝে crash হলে দুটোই rollback → User 1 এর টাকা নিরাপদ
```

---

## 5.2 ACID Properties

| Property | মানে | PostgreSQL এ কীভাবে |
|---|---|---|
| **A**tomicity | সব নয়তো কিছুই না | WAL + CLOG (ABORTED mark) |
| **C**onsistency | সবসময় valid state | CHECK, FK, UNIQUE constraints |
| **I**solation | Transactions একে অপরকে disturb করে না | MVCC + Locks |
| **D**urability | Committed data permanent | WAL synchronous flush |

---

## 5.3 Basic Commands

```sql
BEGIN;                          -- Transaction শুরু
BEGIN ISOLATION LEVEL REPEATABLE READ;  -- Specific isolation level এ শুরু
COMMIT;                         -- সব changes save
ROLLBACK;                       -- সব changes undo

-- Savepoint — partial rollback
BEGIN;
    INSERT INTO orders (user_id, total) VALUES (1, 500);
    SAVEPOINT before_items;
    INSERT INTO order_items (order_id, product_id) VALUES (1, 999); -- Error!
    ROLLBACK TO SAVEPOINT before_items;  -- শুধু এই line undo
    INSERT INTO order_items (order_id, product_id) VALUES (1, 5);   -- Correct
COMMIT;
-- orders insert থাকবে, ভুল order_items undo হবে

RELEASE SAVEPOINT before_items;  -- Savepoint delete করো
```

**Autocommit:**
```sql
-- Default: autocommit ON
-- প্রতিটা statement নিজেই একটা transaction
INSERT INTO users VALUES (...);  -- automatically committed

-- BEGIN দিলে autocommit override হয়
BEGIN;
    INSERT INTO users VALUES (...);  -- এখনো committed না
    -- আরো কাজ করো
COMMIT;  -- এখন committed
```

---

## 5.4 Isolation Levels

**চারটা সমস্যা যা Isolation prevent করে:**

| সমস্যা | মানে | Example |
|---|---|---|
| Dirty Read | Uncommitted data পড়া | B এর uncommitted change A দেখছে |
| Non-Repeatable Read | Same query দুইবার আলাদা result | A দুইবার পড়ল, B মাঝে update করল |
| Phantom Read | Same range query তে নতুন row দেখা | A count করল 5, B insert করল, A আবার count করল 6 |
| Serialization Anomaly | Concurrent এ logical inconsistency | এককভাবে চালালে যা হতো তার চেয়ে আলাদা result |

**PostgreSQL Isolation Levels:**

| Level | Dirty Read | Non-Repeatable | Phantom | Serialization |
|---|---|---|---|---|
| READ COMMITTED (default) | না | হয় | হয় | হয় |
| REPEATABLE READ | না | না | না* | হয় |
| SERIALIZABLE | না | না | না | না |

*PostgreSQL এর REPEATABLE READ এ Phantom Read ও হয় না — MySQL এর চেয়ে strong।

**PostgreSQL তে READ UNCOMMITTED নেই** — minimum READ COMMITTED।

```sql
-- Isolation level দেখো
SHOW transaction_isolation;

-- Session এ change করো
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- অথবা BEGIN এ:
BEGIN ISOLATION LEVEL SERIALIZABLE;

-- postgresql.conf এ default change করো
-- default_transaction_isolation = 'read committed'
```

**SERIALIZABLE — SSI (Serializable Snapshot Isolation):**
```
PostgreSQL এর SERIALIZABLE MySQL এর চেয়ে অনেক smart।
MySQL SERIALIZABLE: সব SELECT এ shared lock → concurrency কমে
PostgreSQL SSI: Predicate lock দিয়ে actual serialization conflict detect করে
               Conflict হলে একটা transaction abort করে (serialization failure)
               No lock, high concurrency
```

---

## 5.5 MVCC Deep Dive

```sql
-- Current Transaction ID
SELECT txid_current();

-- Snapshot দেখো
SELECT txid_current_snapshot();
-- Result: 150:155:151,153
-- xmin=150: এর আগের সব transaction committed (visible)
-- xmax=155: এর পরের সব transaction invisible
-- 151,153:  এরা active (invisible)
-- মানে: 150 এর আগে committed = visible
--       150-155 এর মধ্যে active = invisible
--       155 এবং পরে = invisible

-- Hidden columns দেখো
SELECT xmin, xmax, ctid, id, name FROM users LIMIT 5;
```

---

## 5.6 Lock Types

### Table-Level Locks

```
সবচেয়ে weak → সবচেয়ে strong:

ACCESS SHARE          → SELECT
                         সবকিছুর সাথে compatible (শুধু ACCESS EXCLUSIVE ছাড়া)

ROW SHARE             → SELECT FOR UPDATE / FOR SHARE

ROW EXCLUSIVE         → INSERT, UPDATE, DELETE
                         অন্য ROW EXCLUSIVE এর সাথে compatible (concurrent writes ok)

SHARE UPDATE EXCLUSIVE → VACUUM, ANALYZE, CREATE INDEX CONCURRENTLY
                          Regular DML এর সাথে compatible

SHARE                 → CREATE INDEX (non-concurrent)
                          DML block করে!

ACCESS EXCLUSIVE      → DROP TABLE, TRUNCATE, REINDEX, ALTER TABLE
                          সবকিছু block করে ← সবচেয়ে dangerous
                          Production এ ALTER TABLE → downtime!
```

```sql
-- Explicit table lock (সাধারণত লাগে না, automatic হয়)
LOCK TABLE users IN ACCESS EXCLUSIVE MODE;
LOCK TABLE users IN ROW EXCLUSIVE MODE;
```

### Row-Level Locks

```sql
-- Exclusive row lock — অন্য কেউ modify করতে পারবে না
SELECT * FROM users WHERE id = 5 FOR UPDATE;

-- Shared row lock — read করতে পারবে কিন্তু write করতে পারবে না
SELECT * FROM users WHERE id = 5 FOR SHARE;

-- Lock নেওয়া row skip করো (Job Queue pattern)
SELECT * FROM job_queue
WHERE status = 'pending'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;
-- অন্য worker যেটা নিয়েছে সেটা skip করে পরেরটা নেবে
-- Multiple worker concurrently কাজ করতে পারবে same queue থেকে

-- Lock না পেলে error (wait না করে)
SELECT * FROM users WHERE id = 5 FOR UPDATE NOWAIT;
-- ERROR: could not obtain lock on row in relation "users"
```

---

## 5.7 Deadlock

```
Transaction A:          Transaction B:
UPDATE users id=1 ✅    UPDATE users id=2 ✅
UPDATE users id=2 ⏳    UPDATE users id=1 ⏳
        └──── Deadlock ─────┘

PostgreSQL automatically detect করে এবং একটাকে rollback করে:
ERROR: deadlock detected
DETAIL: Process 12345 waits for ShareLock on transaction 99999
HINT: See server log for query details.
```

**Deadlock এড়ানোর উপায়:**
```sql
-- 1. সবসময় একই order এ rows access করো
-- Always lock id ASC order:
UPDATE accounts SET balance = balance - 100 WHERE id = MIN(1,2);  -- id=1 first
UPDATE accounts SET balance = balance + 100 WHERE id = MAX(1,2);  -- id=2 second

-- 2. Transaction ছোট রাখো — শুধু DB কাজ transaction এর ভেতরে
-- API call, file write → transaction এর বাইরে করো

-- 3. NOWAIT বা SKIP LOCKED ব্যবহার করো
SELECT * FROM inventory WHERE product_id = 10 FOR UPDATE NOWAIT;

-- Log দেখো deadlock debug করতে
-- postgresql.conf: log_lock_waits = on
```

---

## 5.8 Advisory Locks — Application-Level Custom Locks

MySQL তে নেই। PostgreSQL unique feature। Application logic এ exclusive operation guarantee করতে।

```sql
-- Session-level (connection disconnect হলে auto-release)
SELECT pg_advisory_lock(12345);         -- Exclusive lock on key 12345
SELECT pg_advisory_lock_shared(12345);  -- Shared lock
SELECT pg_advisory_unlock(12345);       -- Manual release

-- Transaction-level (transaction end এ auto-release)
SELECT pg_advisory_xact_lock(12345);    -- Transaction end এ release

-- Non-blocking (wait না করে)
SELECT pg_try_advisory_lock(12345);     -- TRUE = got lock, FALSE = already locked

-- Use cases:
-- "শুধু একটা cron job চলুক একসাথে"
-- "নির্দিষ্ট user এর balance update একসাথে একটাই চলুক"

-- Example: Cron job lock
DO $$
BEGIN
    IF pg_try_advisory_lock(42) THEN
        -- Job চালাও
        PERFORM run_monthly_report();
        PERFORM pg_advisory_unlock(42);
    ELSE
        RAISE NOTICE 'Job already running, skipping';
    END IF;
END $$;
```

---

## 5.9 Lock Monitoring

```sql
-- Current locks দেখো
SELECT
    pid,
    locktype,
    relation::regclass AS table_name,
    mode,
    granted,
    LEFT(query, 60) AS query
FROM pg_locks l
JOIN pg_stat_activity a USING (pid)
WHERE relation IS NOT NULL
ORDER BY granted, pid;

-- কোন query কোনটার জন্য অপেক্ষা করছে
SELECT
    blocked.pid         AS blocked_pid,
    blocked.usename     AS blocked_user,
    LEFT(blocked.query, 80)  AS blocked_query,
    blocking.pid        AS blocking_pid,
    blocking.usename    AS blocking_user,
    LEFT(blocking.query, 80) AS blocking_query,
    now() - blocked.xact_start AS waiting_duration
FROM pg_stat_activity AS blocked
JOIN pg_stat_activity AS blocking
    ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE cardinality(pg_blocking_pids(blocked.pid)) > 0;

-- Long running transactions (lock ধরে রাখছে)
SELECT pid, now() - xact_start AS duration, state, LEFT(query, 80)
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY duration DESC
LIMIT 10;

-- Problem process kill করো
SELECT pg_cancel_backend(pid);    -- Query cancel করো, connection রাখো
SELECT pg_terminate_backend(pid); -- Connection kill করো

-- Lock wait timeout set করো
SET lock_timeout = '5s';          -- 5s এর বেশি wait করলে error
SET deadlock_timeout = '1s';      -- Default 1s
```

---

# 6. Query Optimization

## 6.1 Statistics and ANALYZE

Planner সঠিক plan choose করতে statistics দরকার। Statistics outdated হলে wrong plan → slow query।

```sql
-- Statistics দেখো
SELECT
    attname AS column,
    n_distinct,              -- Distinct values: -1 = unique, positive = count
    correlation,             -- Physical order correlation (1=sorted, 0=random)
    most_common_vals,        -- Most common values
    most_common_freqs        -- Their frequencies
FROM pg_stats
WHERE tablename = 'orders'
ORDER BY attname;

-- Statistics update করো
ANALYZE orders;
ANALYZE VERBOSE orders;       -- Progress দেখাবে
ANALYZE;                      -- সব table

-- Specific column এ বেশি accurate statistics
ALTER TABLE orders ALTER COLUMN user_id SET STATISTICS 500;
-- Default 100, max 10000
-- High cardinality, important column এ বাড়াও
ANALYZE orders;
```

---

## 6.2 pg_stat_statements — Built-in Query Tracking

Server start থেকে সব query এর aggregate stats track করে।

```sql
-- Extension load করো (postgresql.conf এ shared_preload_libraries তে যোগ করো)
-- shared_preload_libraries = 'pg_stat_statements'
CREATE EXTENSION pg_stat_statements;

-- Top 10 slowest queries (total execution time দিয়ে)
SELECT
    LEFT(query, 100)                            AS query,
    calls,
    ROUND(total_exec_time::numeric, 2)          AS total_ms,
    ROUND(mean_exec_time::numeric, 2)           AS avg_ms,
    ROUND(stddev_exec_time::numeric, 2)         AS stddev_ms,
    rows,
    ROUND(shared_blks_hit * 100.0 /
        NULLIF(shared_blks_hit + shared_blks_read, 0), 2) AS cache_hit_pct
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- সবচেয়ে বেশিবার চলা queries
SELECT LEFT(query, 80), calls, ROUND(mean_exec_time::numeric, 2) AS avg_ms
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 10;

-- Cache miss বেশি — I/O heavy queries
SELECT LEFT(query, 80), shared_blks_read AS disk_reads
FROM pg_stat_statements
ORDER BY shared_blks_read DESC
LIMIT 10;

-- Statistics reset করো
SELECT pg_stat_statements_reset();
```

---

## 6.3 pg_stat_monitor — Enhanced Query Analytics (Percona)

`pg_stat_statements` এর advanced replacement। Time buckets, execution plan, histogram সহ।

```bash
# Install
sudo dnf install -y pg_stat_monitor_16
# অথবা Percona repo থেকে:
sudo dnf install -y percona-pg_stat_monitor16
```

```ini
# postgresql.conf
shared_preload_libraries = 'pg_stat_monitor'   # pg_stat_statements replace করো

pg_stat_monitor.pgsm_bucket_time        = 60   # 60s time buckets
pg_stat_monitor.pgsm_max_buckets        = 10   # 10 buckets = 10min history
pg_stat_monitor.pgsm_enable_query_plan  = yes  # Execution plan capture করো!
pg_stat_monitor.pgsm_query_max_len      = 2048
pg_stat_monitor.pgsm_histogram_min      = 1
pg_stat_monitor.pgsm_histogram_max      = 100000
pg_stat_monitor.pgsm_histogram_buckets  = 20
```

```sql
CREATE EXTENSION pg_stat_monitor;

-- Slowest queries — এই মুহূর্তে (last bucket)
SELECT
    bucket_start_time                       AS time_window,
    LEFT(query, 100)                        AS query,
    calls,
    ROUND(mean_exec_time::numeric, 2)       AS avg_ms,
    ROUND(min_exec_time::numeric, 2)        AS min_ms,
    ROUND(max_exec_time::numeric, 2)        AS max_ms,
    ROUND(p99_exec_time::numeric, 2)        AS p99_ms,
    rows,
    username,
    datname,
    client_ip
FROM pg_stat_monitor
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Time-based analysis: কোন সময়ে slow হয়েছিল
SELECT
    bucket_start_time,
    calls,
    ROUND(mean_exec_time::numeric, 2) AS avg_ms,
    ROUND(max_exec_time::numeric, 2)  AS max_ms
FROM pg_stat_monitor
WHERE query ILIKE '%orders%'
ORDER BY bucket_start_time DESC;

-- Execution plan দেখো (plan capture ON থাকলে)
SELECT LEFT(query, 80) AS query, calls, ROUND(mean_exec_time::numeric, 2) AS avg_ms, query_plan
FROM pg_stat_monitor
WHERE query_plan IS NOT NULL
ORDER BY mean_exec_time DESC
LIMIT 5;

-- Error-prone queries
SELECT LEFT(query, 80) AS query, calls, errors,
    ROUND(errors * 100.0 / NULLIF(calls, 0), 2) AS error_pct
FROM pg_stat_monitor
WHERE errors > 0
ORDER BY error_pct DESC;

-- Per client IP — কোন app server থেকে slow query
SELECT client_ip, sum(calls) AS total_calls,
    ROUND(avg(mean_exec_time)::numeric, 2) AS avg_ms
FROM pg_stat_monitor
GROUP BY client_ip
ORDER BY total_calls DESC;

SELECT pg_stat_monitor_reset();
```

**pg_stat_statements vs pg_stat_monitor:**

| Feature | pg_stat_statements | pg_stat_monitor |
|---|---|---|
| Basic stats (calls, time, rows) | ✅ | ✅ |
| Time bucket history | ❌ | ✅ "কখন slow ছিল" |
| Execution plan capture | ❌ | ✅ |
| P50/P95/P99 percentiles | ❌ | ✅ |
| Per-client-IP breakdown | ❌ | ✅ |
| Error tracking | ❌ | ✅ |
| PMM QAN integration | Limited | ✅ Native |
| Memory overhead | Low | Medium |

---

## 6.4 EXPLAIN — Full Usage

```sql
-- Basic plan (query execute হয় না)
EXPLAIN SELECT * FROM users WHERE email = 'test@example.com';

-- Actual execution + buffer stats (query execute হয়)
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM users WHERE email = 'test@example.com';

-- JSON format (tools এ useful)
EXPLAIN (ANALYZE, FORMAT JSON) SELECT ...;

-- সব options
EXPLAIN (
    ANALYZE true,    -- Actually run করো
    BUFFERS true,    -- Buffer hit/read দেখাও
    VERBOSE true,    -- Extra details
    COSTS true,      -- Cost estimates
    TIMING true,     -- Per-node timing
    SUMMARY true,    -- Total time summary
    FORMAT text
) SELECT ...;
```

**Scan types:**
```
Index Only Scan  → Index থেকেই data পাওয়া গেছে (heap access নেই) ✅✅
Index Scan       → Index + heap fetch ✅
Bitmap Heap Scan → Multiple matches → bitmap → heap ✅
Seq Scan         → পুরো table পড়া ⚠️ (large table এ index দাও)
```

**Join types:**
```
Nested Loop  → Small outer, small inner → fast for small data
Hash Join    → Large tables, equality join → most common
Merge Join   → Both sides sorted → good for sorted data
```

---

## 6.5 Common Slow Patterns এবং Fix

### Pattern 1 — Full Table Scan

```sql
-- ❌
EXPLAIN SELECT * FROM orders WHERE user_id = 5;
-- Seq Scan on orders (rows=1000000)

-- ✅ Fix
CREATE INDEX idx_orders_user_id ON orders(user_id);
-- Index Scan (rows=47)
```

### Pattern 2 — Function on Indexed Column

```sql
-- ❌ index কাজ করে না — function এর result এ কোনো index নেই
SELECT * FROM users WHERE LOWER(email) = 'alice@example.com';
SELECT * FROM orders WHERE DATE(created_at) = '2024-03-08';
SELECT * FROM users WHERE EXTRACT(YEAR FROM birth_date) = 1990;

-- ✅ Fix: Expression Index দাও
CREATE INDEX idx_lower_email ON users(LOWER(email));
SELECT * FROM users WHERE LOWER(email) = 'alice@example.com';  -- index use করবে

-- ✅ Fix: Range query ব্যবহার করো (date এর জন্য)
SELECT * FROM orders
WHERE created_at >= '2024-03-08'
  AND created_at < '2024-03-09';
```

### Pattern 3 — LIKE Leading Wildcard

```sql
-- ❌ B-Tree index কাজ করে না শুরুতে % থাকলে
SELECT * FROM products WHERE name LIKE '%phone%';

-- ✅ Fix: pg_trgm extension
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_trgm_name ON products USING GIN(name gin_trgm_ops);
SELECT * FROM products WHERE name LIKE '%phone%';   -- এখন index use করবে
SELECT * FROM products WHERE name ILIKE '%PHONE%';  -- case-insensitive ও

-- ✅ Fix: Full-text search (better for large text)
CREATE INDEX idx_fts ON products USING GIN(to_tsvector('english', name));
SELECT * FROM products
WHERE to_tsvector('english', name) @@ to_tsquery('english', 'phone');

-- শুধু prefix search হলে trailing wildcard B-Tree তে কাজ করে:
SELECT * FROM users WHERE username LIKE 'alice%';  -- ✅ index use করবে
```

### Pattern 4 — N+1 Query

```sql
-- ❌ N+1: প্রতিটা user এর জন্য আলাদা query
users = SELECT id FROM users WHERE country = 'BD';  -- 1 query
for each user:
    SELECT COUNT(*) FROM orders WHERE user_id = ?;  -- N queries

-- ✅ Fix: একটা JOIN query
SELECT u.id, u.name, COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.country = 'BD'
GROUP BY u.id, u.name;
-- 1 query, much faster
```

### Pattern 5 — Large OFFSET Pagination

```sql
-- ❌ offset=1000000 মানে 1 million row scan করে শেষ 10টা নেওয়া
SELECT * FROM orders ORDER BY id LIMIT 10 OFFSET 1000000;

-- ✅ Fix: Keyset Pagination (Cursor-based)
-- First page:
SELECT * FROM orders ORDER BY id LIMIT 10;

-- Next page (last seen id=1000000):
SELECT * FROM orders WHERE id > 1000000 ORDER BY id LIMIT 10;
-- → O(log n) instead of O(n)
```

### Pattern 6 — Implicit Type Conversion

```sql
-- ❌ user_id INTEGER column এ string দিলে implicit cast → index নষ্ট
SELECT * FROM users WHERE user_id = '5';     -- '5' is TEXT
SELECT * FROM users WHERE phone = 9876543210;-- phone is TEXT, number দিচ্ছো

-- ✅ Correct type ব্যবহার করো
SELECT * FROM users WHERE user_id = 5;
SELECT * FROM users WHERE phone = '9876543210';
```

### Pattern 7 — Correlated Subquery

```sql
-- ❌ প্রতিটা row এর জন্য subquery চলে
SELECT * FROM users u
WHERE (SELECT COUNT(*) FROM orders o WHERE o.user_id = u.id) > 5;

-- ✅ Fix: JOIN দিয়ে
SELECT u.*
FROM users u
INNER JOIN (
    SELECT user_id FROM orders
    GROUP BY user_id
    HAVING COUNT(*) > 5
) active_users ON u.id = active_users.user_id;
```

---

## 6.6 Planner Settings Override

```sql
-- Debug: Index দেওয়ার পরও Seq Scan হলে
SET enable_seqscan = off;    -- Force index scan
EXPLAIN SELECT ...;
SET enable_seqscan = on;     -- Reset

-- pg_hint_plan extension (production এ index force করতে)
CREATE EXTENSION pg_hint_plan;
/*+ IndexScan(users users_email_idx) */
SELECT * FROM users WHERE LOWER(email) = 'alice@example.com';
```

---

## 6.7 Table Partitioning

Large table কে ছোট physical parts এ ভাগ করা। Query শুধু relevant partition scan করে।

```sql
-- Range Partitioning (time-series, logs এ সবচেয়ে common)
CREATE TABLE orders (
    id         BIGINT GENERATED ALWAYS AS IDENTITY,
    user_id    INTEGER,
    total      NUMERIC(10,2),
    created_at TIMESTAMPTZ NOT NULL
) PARTITION BY RANGE (created_at);

-- Partitions তৈরি করো
CREATE TABLE orders_2023 PARTITION OF orders
    FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');
CREATE TABLE orders_2024 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

-- Default partition (কোনটাতেই না পড়লে)
CREATE TABLE orders_default PARTITION OF orders DEFAULT;

-- List Partitioning
CREATE TABLE users_global (
    id      BIGINT,
    country TEXT NOT NULL
) PARTITION BY LIST (country);

CREATE TABLE users_bd PARTITION OF users_global FOR VALUES IN ('BD');
CREATE TABLE users_in PARTITION OF users_global FOR VALUES IN ('IN');
CREATE TABLE users_other PARTITION OF users_global DEFAULT;

-- Hash Partitioning (evenly distribute)
CREATE TABLE events (id BIGINT, data JSONB)
    PARTITION BY HASH (id);
CREATE TABLE events_0 PARTITION OF events FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE events_1 PARTITION OF events FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE events_2 PARTITION OF events FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE events_3 PARTITION OF events FOR VALUES WITH (MODULUS 4, REMAINDER 3);

-- Partition Pruning দেখো
EXPLAIN SELECT * FROM orders WHERE created_at >= '2024-01-01';
-- শুধু orders_2024 scan করবে → orders_2023 skip (Partition Pruning)

-- Index — parent table এ দিলে সব partition এ automatically
CREATE INDEX idx_orders_user ON orders(user_id);
CREATE INDEX idx_orders_created ON orders(created_at DESC);
```

---

## 6.8 CTEs (Common Table Expressions)

```sql
-- Basic CTE — পড়তে সহজ, complex query simplify করে
WITH active_users AS (
    SELECT id, name, country FROM users WHERE status = 'active'
),
user_order_counts AS (
    SELECT user_id, COUNT(*) AS order_count, SUM(total) AS total_spent
    FROM orders
    WHERE created_at > NOW() - INTERVAL '1 year'
    GROUP BY user_id
)
SELECT u.name, u.country,
    COALESCE(uo.order_count, 0) AS orders,
    COALESCE(uo.total_spent, 0) AS spent
FROM active_users u
LEFT JOIN user_order_counts uo ON u.id = uo.user_id
ORDER BY spent DESC;

-- Recursive CTE — Tree/hierarchy traverse
WITH RECURSIVE category_tree AS (
    -- Base case: root categories
    SELECT id, name, parent_id, 0 AS depth, name AS path
    FROM categories WHERE parent_id IS NULL

    UNION ALL

    -- Recursive case: children
    SELECT c.id, c.name, c.parent_id, ct.depth + 1,
        ct.path || ' > ' || c.name
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT depth, path FROM category_tree ORDER BY path;

-- MATERIALIZED vs NOT MATERIALIZED (PostgreSQL 12+)
WITH data AS MATERIALIZED (
    SELECT * FROM large_table WHERE expensive_condition = true
)
SELECT * FROM data WHERE another_filter = 'x';
-- MATERIALIZED: CTE result একবার compute করে store করে
-- NOT MATERIALIZED: Inline করে (optimizer plan করতে পারে)
-- Default: PostgreSQL decide করে
```

---

## 6.9 Parallel Query

```sql
-- Parallel query enable আছে কিনা
SHOW max_parallel_workers_per_gather;  -- Default: 2

-- Large table scan এ parallel automatically হয়
EXPLAIN SELECT COUNT(*) FROM large_events_table;
-- Gather (2 workers)
--   -> Parallel Seq Scan on large_events_table

-- Disable করো (debug এর জন্য)
SET max_parallel_workers_per_gather = 0;

-- Specific query তে hint
SET max_parallel_workers_per_gather = 8;
SELECT COUNT(*) FROM huge_table;
RESET max_parallel_workers_per_gather;
```

---

## 6.10 Optimization Checklist

```
Query লেখার সময়:
  ✅ SELECT * এড়াও — দরকারি column specify করো
  ✅ WHERE column এ function ব্যবহার করো না (Expression Index দাও)
  ✅ Correct data type দিয়ে compare করো (implicit cast এড়াও)
  ✅ N+1 query এড়াও → JOIN ব্যবহার করো
  ✅ Large OFFSET এড়াও → Keyset Pagination
  ✅ Correlated Subquery → JOIN এ convert করো

Index দেওয়ার সময়:
  ✅ WHERE, JOIN column → B-Tree
  ✅ ORDER BY column → B-Tree (DESC যদি দরকার হয়)
  ✅ JSONB, Array → GIN
  ✅ Range type → GiST
  ✅ Sequential/Time data → BRIN
  ✅ Low cardinality filter → Partial Index
  ✅ Function on column → Expression Index
  ✅ Index + extra columns → Covering Index (INCLUDE)
  ✅ Unused index DROP করো

Verify করার সময়:
  ✅ EXPLAIN (ANALYZE, BUFFERS) চালাও
  ✅ Seq Scan on large table নেই
  ✅ Shared Buffer hit rate 99%+
  ✅ Statistics up-to-date (ANALYZE)
  ✅ Table bloat নেই (VACUUM চলছে)
```

---

## 6.11 JOIN Optimization — Deep Dive

### তিনটা JOIN Algorithm

PostgreSQL তিনটা আলাদা algorithm দিয়ে JOIN করতে পারে। Planner data size দেখে সবচেয়ে efficient টা choose করে।

```
Nested Loop Join:
  for each row in outer:
      for each row in inner:
          if match: output
  
  Best for: Small tables, indexed inner table
  Cost: O(n × m) — outer × inner
  
  Example:
    users table: 100 rows (outer)
    orders table: index on user_id
    → 100 index lookups on orders → fast

Hash Join:
  Phase 1 (Build):   Smaller table → Hash Table তৈরি (memory তে)
  Phase 2 (Probe):   Larger table scan করো, hash তে match করো
  
  Best for: Large tables, equality join (=)
  Memory: work_mem দিয়ে নিয়ন্ত্রিত
  Hash table memory তে না ধরলে → disk spill → slow
  
  Example:
    orders: 1M rows, users: 10K rows
    → users দিয়ে hash table (10K) build
    → orders scan করে hash match

Merge Join:
  দুটো input sorted হলে → linear merge
  
  Best for: Already sorted data, large sorted indexes
  Cost: O(n + m) — খুব fast
  Sort cost থাকে যদি already sorted না হয়
```

```sql
-- JOIN type force করো (debug এর জন্য)
SET enable_nestloop = off;   -- Nested Loop disable
SET enable_hashjoin = off;   -- Hash Join disable
SET enable_mergejoin = off;  -- Merge Join disable

-- EXPLAIN এ JOIN type দেখো
EXPLAIN (ANALYZE, BUFFERS)
SELECT u.name, COUNT(o.id)
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.country = 'BD'
GROUP BY u.name;

-- Hash Join কতটা memory নিয়েছে
-- EXPLAIN এ দেখো: Batches: 1 → memory তে fit করেছে
--                 Batches: 4 → 4x disk spill হয়েছে
-- Batches > 1 হলে work_mem বাড়াও
```

### JOIN Performance Rules

```sql
-- Rule 1: JOIN column এ সবসময় index দাও
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_order_items_order_id ON order_items(order_id);

-- Rule 2: Smaller table কে LEFT এ রাখো (Nested Loop এ outer হয়)
-- Planner usually সঠিক করে, কিন্তু statistics outdated হলে wrong হতে পারে

-- Rule 3: JOIN condition এ function এড়াও
-- ❌
JOIN users u ON LOWER(o.email) = LOWER(u.email)
-- ✅ Expression index দাও:
CREATE INDEX idx_lower_email ON users(LOWER(email));
CREATE INDEX idx_lower_order_email ON orders(LOWER(email));
JOIN users u ON LOWER(o.email) = LOWER(u.email)  -- এখন index use করবে

-- Rule 4: Many-to-many JOIN এ intermediate table এ index দাও
-- user_roles(user_id, role_id) → দুটোতেই index দাও
CREATE INDEX idx_user_roles_user ON user_roles(user_id);
CREATE INDEX idx_user_roles_role ON user_roles(role_id);

-- Rule 5: Subquery vs JOIN — JOIN prefer করো
-- ❌ Subquery
SELECT * FROM orders WHERE user_id IN (SELECT id FROM users WHERE country = 'BD');
-- ✅ JOIN (planner often converts automatically, কিন্তু explicit better)
SELECT o.* FROM orders o
JOIN users u ON o.user_id = u.id
WHERE u.country = 'BD';

-- Rule 6: CROSS JOIN এড়াও (accidental বা intentional)
-- বিপদজনক: N×M rows তৈরি হয়
-- 1000 × 1000 = 1,000,000 rows!
-- সবসময় JOIN condition দাও
```

### Statistics এর কারণে Wrong Join Plan

```sql
-- Wrong row estimate → Wrong join order → Slow query
-- পরীক্ষা করো:
EXPLAIN (ANALYZE)
SELECT ... FROM orders o JOIN users u ON o.user_id = u.id WHERE ...;

-- "rows=X (actual rows=Y)" — X এবং Y অনেক আলাদা হলে statistics outdated
-- Fix:
ANALYZE orders;
ANALYZE users;

-- Specific column এ বেশি statistics দাও
ALTER TABLE orders ALTER COLUMN user_id SET STATISTICS 500;
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 200;
ANALYZE orders;

-- pg_hint_plan দিয়ে join order force করো (last resort)
CREATE EXTENSION pg_hint_plan;
/*+ Leading(u o) HashJoin(u o) */
SELECT u.name, COUNT(o.id) FROM users u JOIN orders o ON u.id = o.user_id ...;
```

---

## 6.12 Aggregate এবং GROUP BY Optimization

```sql
-- HashAggregate vs GroupAggregate
-- HashAggregate: Hash table তৈরি করে group করে → fast, work_mem দরকার
-- GroupAggregate: Sorted input লাগে → sort cost + sequential merge → slower

EXPLAIN SELECT status, COUNT(*) FROM orders GROUP BY status;
-- HashAggregate (fast, unordered output)

EXPLAIN SELECT status, COUNT(*) FROM orders GROUP BY status ORDER BY status;
-- GroupAggregate বা Sort + HashAggregate

-- HashAggregate এ batches > 1 হলে work_mem বাড়াও:
SET work_mem = '256MB';
EXPLAIN (ANALYZE) SELECT category_id, COUNT(*), SUM(total) FROM orders GROUP BY category_id;
-- Batches: 1 → memory তে fit
```

### Common Aggregate Patterns

```sql
-- COUNT(*) vs COUNT(column)
COUNT(*)       -- সব rows গণনা করে (NULL সহ) — fastest
COUNT(id)      -- Non-NULL id গণনা করে
COUNT(DISTINCT user_id)  -- Unique count — expensive (sort বা hash লাগে)

-- DISTINCT এর বদলে GROUP BY তে aggregate
-- ❌ Slow
SELECT DISTINCT user_id FROM orders WHERE total > 100;
-- ✅ Same result, usually faster planner options
SELECT user_id FROM orders WHERE total > 100 GROUP BY user_id;

-- Filtered Aggregate (FILTER clause)
SELECT
    COUNT(*) AS total_orders,
    COUNT(*) FILTER (WHERE status = 'completed') AS completed,
    COUNT(*) FILTER (WHERE status = 'pending')   AS pending,
    SUM(total) FILTER (WHERE status = 'completed') AS completed_revenue
FROM orders;
-- একটাই query, একটাই table scan — multiple WHERE এর চেয়ে fast

-- Window Function vs GROUP BY
-- "প্রতিটা order সহ সেই user এর total order count"
-- ❌ Correlated subquery (N+1)
SELECT o.*,
    (SELECT COUNT(*) FROM orders WHERE user_id = o.user_id) AS user_order_count
FROM orders o;
-- ✅ Window function (একটাই scan)
SELECT o.*,
    COUNT(*) OVER (PARTITION BY user_id) AS user_order_count
FROM orders o;

-- Running total
SELECT id, total,
    SUM(total) OVER (PARTITION BY user_id ORDER BY created_at) AS running_total
FROM orders;

-- Rank
SELECT id, user_id, total,
    RANK() OVER (PARTITION BY user_id ORDER BY total DESC) AS rank_by_total
FROM orders;
-- এই user এর কত নম্বর বড় order

-- Window function এ Index:
-- PARTITION BY column এ index → partition faster
-- ORDER BY column এ index → sort avoid করতে পারে
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at);
```

### Materialized Views — Expensive Aggregate Pre-compute

```sql
-- Expensive dashboard query যেটা বারবার চলে
-- প্রতিবার চালানো slow → Pre-compute করে রাখো

CREATE MATERIALIZED VIEW daily_sales_summary AS
SELECT
    DATE(created_at)  AS sale_date,
    COUNT(*)          AS order_count,
    SUM(total)        AS total_revenue,
    AVG(total)        AS avg_order_value,
    COUNT(DISTINCT user_id) AS unique_buyers
FROM orders
WHERE status = 'completed'
GROUP BY DATE(created_at)
ORDER BY sale_date;

CREATE UNIQUE INDEX idx_daily_sales_date ON daily_sales_summary(sale_date);

-- Query — instant (precomputed)
SELECT * FROM daily_sales_summary WHERE sale_date >= NOW() - INTERVAL '30 days';

-- Refresh (নতুন data আনতে)
REFRESH MATERIALIZED VIEW daily_sales_summary;
-- সব data recompute করে — কিছুক্ষণ লাগে

-- CONCURRENTLY — refresh এর সময়ও read allow (unique index দরকার)
REFRESH MATERIALIZED VIEW CONCURRENTLY daily_sales_summary;

-- Cron এ auto-refresh করো:
-- 0 * * * * psql -c "REFRESH MATERIALIZED VIEW CONCURRENTLY daily_sales_summary;"

-- Regular View vs Materialized View
-- Regular VIEW:     Query চালানোর সময় compute হয় (stored data নেই)
-- Materialized VIEW: Disk এ stored (pre-computed), REFRESH এ update হয়
-- ব্যবহার করো: heavy aggregate, dashboard query, reporting
```

---

## 6.13 Write Performance (INSERT/UPDATE/DELETE)

### Bulk INSERT — সবচেয়ে Important

```sql
-- ❌ Row-by-row INSERT — অনেক slow
for each row:
    INSERT INTO orders VALUES (...);
-- প্রতিটা INSERT = separate transaction, separate WAL write

-- ✅ Multi-row INSERT
INSERT INTO orders (user_id, total, status)
VALUES (1, 500, 'pending'),
       (2, 750, 'pending'),
       (3, 200, 'completed');
-- একটা WAL write, একটা transaction

-- ✅ COPY — সবচেয়ে fast (bulk load এ)
COPY orders (user_id, total, status) FROM '/tmp/orders.csv' CSV HEADER;
-- INSERT এর চেয়ে 5-10x fast
-- WAL এ কম লেখে, direct heap insert
-- COPY FROM stdin (application থেকে):
\copy orders FROM 'orders.csv' CSV HEADER;  -- psql এ

-- ✅ INSERT ... SELECT — table থেকে table
INSERT INTO orders_archive SELECT * FROM orders WHERE created_at < '2023-01-01';

-- Large bulk insert এর জন্য optimization:
-- 1. Index temporarily drop করো (যদি empty table)
DROP INDEX idx_orders_user_id;
COPY orders FROM '/tmp/orders.csv' CSV;
CREATE INDEX idx_orders_user_id ON orders(user_id);
-- Index একবারে build করা অনেক fast

-- 2. WAL কমাও (data loss acceptable হলে)
SET synchronous_commit = off;
BEGIN;
    INSERT ... (large batch)
COMMIT;
SET synchronous_commit = on;

-- 3. UNLOGGED TABLE — WAL write নেই, crash হলে হারায়
CREATE UNLOGGED TABLE import_staging (...);
COPY import_staging FROM '/tmp/data.csv' CSV;
-- Process করো
INSERT INTO real_table SELECT ... FROM import_staging;
DROP TABLE import_staging;
```

### UPSERT — INSERT or UPDATE

```sql
-- ON CONFLICT দিয়ে
INSERT INTO users (id, email, name, updated_at)
VALUES (5, 'alice@example.com', 'Alice', NOW())
ON CONFLICT (id) DO UPDATE
    SET email      = EXCLUDED.email,
        name       = EXCLUDED.name,
        updated_at = NOW();
-- EXCLUDED = নতুন values যেগুলো insert হতো

-- Specific column এ conflict
INSERT INTO user_settings (user_id, key, value)
VALUES (5, 'theme', 'dark')
ON CONFLICT (user_id, key) DO UPDATE
    SET value = EXCLUDED.value;

-- Conflict হলে ignore করো (do nothing)
INSERT INTO events (id, data)
VALUES (123, '...')
ON CONFLICT (id) DO NOTHING;

-- Performance: ON CONFLICT DO UPDATE তে
-- conflict column এ index থাকা mandatory (UNIQUE constraint)
-- EXCLUDED pseudo-table use করো (subquery এড়াতে)
```

### UPDATE Performance

```sql
-- ❌ Unnecessary UPDATE এড়াও
UPDATE users SET last_seen = NOW();  -- সব row update!

-- ✅ WHERE দিয়ে filter করো
UPDATE users SET last_seen = NOW() WHERE id = 5;

-- ❌ UPDATE এ subquery
UPDATE orders SET status = 'shipped'
WHERE user_id IN (SELECT id FROM users WHERE country = 'BD');
-- ✅ UPDATE ... FROM (faster)
UPDATE orders o
SET status = 'shipped'
FROM users u
WHERE o.user_id = u.id AND u.country = 'BD';

-- Batch UPDATE (large table এ একসাথে অনেক update → bloat হয়)
-- ❌ একবারে সব
UPDATE large_table SET column = value WHERE condition;  -- millions of rows!

-- ✅ Batch করো (autovacuum কে সুযোগ দাও)
DO $$
DECLARE
    batch_size INT := 10000;
    updated INT;
BEGIN
    LOOP
        UPDATE large_table
        SET column = value
        WHERE id IN (
            SELECT id FROM large_table
            WHERE column != value AND column IS NOT NULL
            LIMIT batch_size
            FOR UPDATE SKIP LOCKED
        );
        GET DIAGNOSTICS updated = ROW_COUNT;
        EXIT WHEN updated = 0;
        PERFORM pg_sleep(0.1);  -- Autovacuum কে সুযোগ দাও
        RAISE NOTICE 'Updated %', updated;
    END LOOP;
END $$;
```

### DELETE Performance

```sql
-- DELETE vs TRUNCATE
DELETE FROM temp_data;           -- Row by row, WAL heavy, slow
TRUNCATE temp_data;              -- Instant, minimal WAL, table reset
TRUNCATE temp_data RESTART IDENTITY; -- Sequence ও reset

-- Batch DELETE (large table এ)
DO $$
DECLARE
    deleted INT;
BEGIN
    LOOP
        DELETE FROM logs
        WHERE id IN (
            SELECT id FROM logs
            WHERE created_at < NOW() - INTERVAL '90 days'
            LIMIT 10000
        );
        GET DIAGNOSTICS deleted = ROW_COUNT;
        EXIT WHEN deleted = 0;
        PERFORM pg_sleep(0.05);  -- IO relief
        RAISE NOTICE 'Deleted %', deleted;
    END LOOP;
END $$;

-- Partitioned table এ DELETE — instant!
-- Partition DROP করো DELETE এর বদলে
DROP TABLE orders_2020;  -- orders PARTITION OF orders_2020 → instant!
-- এটাই time-series data এ partitioning এর সবচেয়ে বড় সুবিধা
```

### RETURNING Clause — Extra Round Trip বাঁচাও

```sql
-- INSERT করার পর ID নিতে আলাদা SELECT করতে হয় না
INSERT INTO users (name, email)
VALUES ('Alice', 'alice@example.com')
RETURNING id, created_at;

-- UPDATE করার পর updated data নাও
UPDATE orders SET status = 'shipped', updated_at = NOW()
WHERE id = 5
RETURNING id, status, updated_at;

-- DELETE করা rows এর data নাও
DELETE FROM expired_sessions
WHERE expires_at < NOW()
RETURNING user_id, session_token;
```

---

## 6.14 Connection Pool Performance

### কেন Connection Pool দরকার

```
PostgreSQL Process Model:
  প্রতিটা connection → fork() → নতুন OS process
  Connection overhead:
    fork() time: ~1-5ms
    Memory: ~5-10MB per process
    PostgreSQL startup: ~20ms

Without pgBouncer:
  1000 connections × 10MB = 10GB শুধু connections
  High connection churn → constant fork/exit overhead
  PostgreSQL max practical: ~200-300 connections

With pgBouncer (transaction pool):
  App: 1000 connections → pgBouncer: 20 backend connections
  Backend connections persist → no fork overhead
  Memory: 20 × 10MB = 200MB
```

### pgBouncer Pool Mode Selection

```
session mode:
  Client lifetime = backend connection lifetime
  App session variables (SET), LISTEN/NOTIFY safe
  Low pool efficiency — জনপ্রিয় না

transaction mode (recommended):
  Backend connection = transaction duration
  Transaction শেষে connection pool এ ফেরত যায়
  High efficiency: 1000 clients, 20 backends
  ⚠️ SET commands persist করে না (next transaction নতুন backend পেতে পারে)
  ⚠️ Advisory locks, LISTEN/NOTIFY কাজ করে না
  → 95% workload এর জন্য এটাই ব্যবহার করো

statement mode:
  Statement শেষেই release
  Autocommit only, multi-statement transaction কাজ করে না
  Rarely used
```

```ini
# pgBouncer tuning for PostgreSQL
[pgbouncer]
pool_mode              = transaction
default_pool_size      = 20        # Per database per user backend connections
max_client_conn        = 1000      # Max app connections
reserve_pool_size      = 5         # Emergency extra connections
reserve_pool_timeout   = 3         # 3s wait করে reserve pool নেয়

server_idle_timeout    = 600       # 10 min idle backend connection close
client_idle_timeout    = 0         # App connection idle timeout (0=never)
server_lifetime        = 3600      # 1 hour এর পর backend connection close/reopen

# Performance
max_db_connections     = 100       # Database এ maximum
tcp_keepalive          = 1
```

```sql
-- pgBouncer stats দেখো
-- psql -p 6432 -U admin pgbouncer
SHOW POOLS;
-- pool_mode, cl_active, cl_waiting, sv_active, sv_idle, sv_used

-- cl_waiting > 0 → সব backend busy, clients wait করছে
-- → default_pool_size বাড়াও বা query optimize করো

SHOW STATS;
-- total_requests, total_received, total_sent, avg_req, avg_query

SHOW CLIENTS;
-- কোন client কতক্ষণ connected, কতটা request করেছে
```

### Application-Side Connection Pool

```python
# Python (asyncpg / psycopg3 pool)
import asyncpg

pool = await asyncpg.create_pool(
    'postgresql://user:pass@localhost/mydb',
    min_size=5,
    max_size=20,  # PostgreSQL এ maximum 20 connections
    max_inactive_connection_lifetime=300,
)

# pgBouncer address দাও (direct PG নয়)
pool = await asyncpg.create_pool(
    'postgresql://user:pass@localhost:6432/mydb',  # pgBouncer port
    min_size=10,
    max_size=100,  # pgBouncer handle করবে
)
```

```java
// Java HikariCP (most popular Java pool)
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:postgresql://localhost:6432/mydb");  // pgBouncer
config.setMaximumPoolSize(50);
config.setMinimumIdle(10);
config.setConnectionTimeout(5000);    // 5s
config.setIdleTimeout(300000);        // 5min
config.setMaxLifetime(1800000);       // 30min

// pgBouncer transaction mode তে:
config.setAutoCommit(false);  // সবসময় explicit transaction
```

---

## 6.15 Autovacuum Performance Tuning

Autovacuum slow হলে বা না চললে → dead tuples জমে → table bloat → query slow।

### কীভাবে Autovacuum Trigger হয়

```
Trigger condition:
  n_dead_tup > autovacuum_vacuum_threshold + autovacuum_vacuum_scale_factor × n_live_tup

Default:
  50 + 0.02 × n_live_tup = dead_tuples

1,000 rows table:  50 + 0.02 × 1,000     = 70 dead tuples trigger
100,000 rows:      50 + 0.02 × 100,000   = 2,050 dead tuples trigger
1,000,000 rows:    50 + 0.02 × 1,000,000 = 20,050 dead tuples trigger
10,000,000 rows:   50 + 0.02 × 10,000,000 = 200,050 dead tuples!

সমস্যা: বড় table এ অনেক dead tuple জমার পর trigger হয়
→ একবারে অনেক কাজ করতে হয় → I/O spike → query slow হয়
→ Scale factor কমাও বড় table এ
```

```sql
-- Per-table autovacuum tune (বড় table এর জন্য aggressive করো)
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor   = 0.005,  -- 0.5% (default 2%)
    autovacuum_vacuum_threshold      = 100,
    autovacuum_analyze_scale_factor  = 0.002,
    autovacuum_analyze_threshold     = 50,
    autovacuum_vacuum_cost_delay     = 2ms     -- I/O throttle
);

-- High-write table (যেমন: session, audit log) এ
ALTER TABLE sessions SET (
    autovacuum_vacuum_scale_factor = 0.001,   -- 0.1%
    autovacuum_vacuum_cost_delay   = 0        -- No throttle, run fast
);

-- Table autovacuum settings দেখো
SELECT relname, reloptions
FROM pg_class
WHERE reloptions IS NOT NULL AND relkind = 'r';
```

```ini
# postgresql.conf — Global tuning

autovacuum_max_workers           = 5     # Default 3, বাড়াও বড় database এ
autovacuum_naptime               = 30s   # Check frequency (default 1min)

# Cost-based throttle — Autovacuum IO কতটা নেবে
autovacuum_vacuum_cost_delay     = 2ms   # প্রতি cost_limit এর পর pause
autovacuum_vacuum_cost_limit     = 400   # Per worker cost limit per round
                                          # Default 200 — বাড়াও faster vacuum এর জন্য
vacuum_cost_page_hit             = 1     # Shared buffer hit cost
vacuum_cost_page_miss            = 2     # Disk read cost (default 10!)
vacuum_cost_page_dirty           = 20    # Dirty page write cost

# কীভাবে tune করবো:
# autovacuum_vacuum_cost_limit = 800  (বাড়ালে faster কিন্তু more I/O)
# autovacuum_vacuum_cost_delay = 2ms  (কমালে faster)
# এই দুটো মিলে: throughput = cost_limit / cost_delay = 800/2 = 400 pages/ms
```

### Autovacuum Monitoring

```sql
-- কোন table এ autovacuum চলছে এখন
SELECT p.pid, relid::regclass AS table_name, phase,
    heap_blks_total, heap_blks_scanned,
    ROUND(heap_blks_scanned * 100.0 / NULLIF(heap_blks_total, 0), 2) AS pct,
    num_dead_tuples
FROM pg_stat_progress_vacuum p
JOIN pg_stat_activity a ON p.pid = a.pid;

-- কোন table autovacuum miss করছে (দেরিতে vacuum হচ্ছে)
SELECT relname,
    n_dead_tup,
    n_live_tup,
    last_autovacuum,
    now() - last_autovacuum AS since_vacuum,
    autovacuum_count
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC
LIMIT 10;

-- Autovacuum কতটা কাজ করছে (last hour)
SELECT schemaname, relname,
    autovacuum_count,
    autoanalyze_count,
    n_dead_tup
FROM pg_stat_user_tables
WHERE autovacuum_count > 0
ORDER BY autovacuum_count DESC
LIMIT 10;

-- autovacuum blocked? কে block করছে?
SELECT pid, now() - xact_start AS duration, state, LEFT(query, 80)
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY duration DESC LIMIT 5;
-- idle in transaction → autovacuum ওই table touch করতে পারছে না!
```

---

## 6.16 VACUUM এবং Table Bloat Management

### Table Bloat কেন হয়

```
UPDATE users SET salary = salary + 1000;  -- 1 million rows update

PostgreSQL এ UPDATE = DELETE old + INSERT new
→ Old rows dead tuple হয় (1 million dead tuples)
→ Autovacuum না চললে এরা table এ থাকে
→ Table size বাড়তে থাকে (bloat)
→ Sequential scan slow (পুরো bloated table পড়তে হয়)
→ Index ও bloated হয়

Bloat = Physical table size - Actual data size
       = "Empty space" যা dead tuples থেকে freed কিন্তু OS কে ফেরত দেওয়া হয়নি
```

### Bloat Measurement

```sql
-- Table bloat estimate (approximate)
WITH table_bloat AS (
    SELECT
        schemaname, tablename,
        pg_relation_size(schemaname||'.'||tablename) AS actual_size,
        pg_total_relation_size(schemaname||'.'||tablename) AS total_size,
        n_dead_tup,
        n_live_tup,
        CASE WHEN n_live_tup + n_dead_tup > 0
            THEN ROUND(n_dead_tup * 100.0 / (n_live_tup + n_dead_tup), 2)
            ELSE 0
        END AS dead_pct
    FROM pg_stat_user_tables
)
SELECT
    schemaname, tablename,
    pg_size_pretty(actual_size) AS table_size,
    dead_pct                    AS bloat_pct,
    n_dead_tup,
    n_live_tup
FROM table_bloat
WHERE actual_size > 1 * 1024 * 1024  -- 1MB+
ORDER BY dead_pct DESC NULLS LAST
LIMIT 20;

-- Index bloat দেখো (btree_bloat function থেকে, pgstattuple extension)
CREATE EXTENSION pgstattuple;

SELECT * FROM pgstattuple('orders');
-- table_len: actual physical size
-- dead_tuple_count, dead_tuple_percent ← bloat %
-- free_space, free_percent ← available space in pages

SELECT * FROM pgstatindex('idx_orders_user_id');
-- leaf_fragmentation: index এর page fragmentation %
-- avg_leaf_density: pages কতটা full (low = bloat)
```

### Bloat Fix করার উপায়

```sql
-- Option 1: VACUUM (dead tuples সরায়, কিন্তু OS কে space ফেরত দেয় না)
VACUUM VERBOSE orders;
-- space table এর পেছনের pages থেকে সরালে OS কে ফেরত দেয়
-- Middle pages এর space → free space map এ mark হয় (future INSERT এ reuse)

-- Option 2: VACUUM FULL (পুরো table rewrite)
VACUUM FULL orders;  -- ⚠️ ACCESS EXCLUSIVE lock — production downtime!
-- Pros: Disk space OS কে ফেরত দেয়, table shrinks
-- Cons: Full table lock, replication lag হতে পারে
-- Production এ এটা ব্যবহার করো না during peak hours

-- Option 3: pg_repack (online, no lock) ✅ Production preferred
pg_repack -h localhost -U postgres -d mydb -t orders
-- Background এ নতুন table তৈরি করে, data copy করে, atomically swap করে
-- Application চলতে থাকে, minimal lock (microseconds)
-- Extension install করতে হবে: CREATE EXTENSION pg_repack;

-- Option 4: CLUSTER (sort করে rewrite — index order অনুযায়ী)
CLUSTER orders USING idx_orders_created_at;
-- Table কে created_at index এর order এ physically sort করে rewrite করে
-- Range scan on created_at এর পর অনেক fast
-- Lock নেয়, production off-peak এ করো
-- CONCURRENTLY version নেই, pg_repack দিয়ে করা যায়

-- Partition এ bloat → Old partition DROP করো (instant, lock নেই)
DROP TABLE orders_2020;  -- orders এর PARTITION → instant!
```

### pg_repack — Detailed Usage

```bash
# Install
sudo dnf install -y pg_repack_16
# postgresql.conf এ:
# shared_preload_libraries = 'pg_repack'

psql -c "CREATE EXTENSION pg_repack;"

# Single table
pg_repack -h localhost -U postgres -d mydb -t orders

# All tables in a database
pg_repack -h localhost -U postgres -d mydb

# Index only repack (faster)
pg_repack -h localhost -U postgres -d mydb --only-indexes -t orders

# Dry run (কী করবে দেখো)
pg_repack -h localhost -U postgres -d mydb -t orders --dry-run

# Time/Size limit
pg_repack -h localhost -U postgres -d mydb \
    --wait-timeout=60 \     # 60s lock wait
    -t orders
```

---

## 6.17 Hardware-Level Tuning

### Storage Optimization

```ini
# postgresql.conf — Storage settings

# SSD এর জন্য
random_page_cost         = 1.1   # Default 4.0 (HDD) → 1.1 (SSD)
effective_io_concurrency = 200   # Default 1 → 200 (SSD parallel I/O)

# OS Page Cache efficient ব্যবহার করতে
effective_cache_size     = 12GB  # Total RAM এর 75% estimate করো
                                  # Planner এই hint দিয়ে index scan prefer করে

# Checkpoint tuning — I/O smooth করতে
checkpoint_completion_target = 0.9
# Checkpoint 90% of timeout time ধরে spread করো
# Spike I/O কমায়, smooth করে
# Default: 0.9 (already good)

max_wal_size = 2GB      # Checkpoint frequency কমাতে (বড় → কম checkpoint → কম I/O)
min_wal_size = 512MB

# fsync — production এ সবসময় on
fsync                = on    # কখনো off করো না (data corruption risk)
full_page_writes     = on    # Partial page write protection

# Write pattern
wal_writer_delay     = 200ms    # WAL flush interval
commit_delay         = 0        # Group commit delay (OLTP এ 0 ভালো)
commit_siblings      = 5        # Group commit min concurrent transactions
```

### Memory Optimization

```ini
# Huge Pages (Linux)
# Large shared memory → OS huge pages → TLB miss কমে → faster

# OS এ enable করো:
# sysctl vm.nr_hugepages = 2048  (2GB huge pages for 4GB shared_buffers)
# echo 'vm.nr_hugepages = 2048' >> /etc/sysctl.conf
# sysctl -p

# postgresql.conf:
huge_pages = try    # try → available হলে use, না হলে regular
# on  → required (না পেলে start fail)
# off → don't use

# Check করো:
# grep -i hugepage /proc/meminfo
# AnonHugePages: ...
# HugePages_Total: 2048
# HugePages_Free: ...

# Shared memory limit OS এ বাড়াও:
# /etc/sysctl.conf:
# kernel.shmmax = 17179869184   (16GB)
# kernel.shmall = 4194304       (16GB in pages)
```

### OS-Level Tuning

```bash
# Swappiness কমাও — PostgreSQL কে RAM এ রাখো
echo 'vm.swappiness = 1' >> /etc/sysctl.conf
# Default 60 → OS aggressively swap করে
# 1 → শুধু OOM এর আগে swap
sysctl -p

# Dirty page settings (faster write)
echo 'vm.dirty_background_ratio = 5' >> /etc/sysctl.conf
echo 'vm.dirty_ratio = 30' >> /etc/sysctl.conf
# dirty_background_ratio: background flush শুরু এই % dirty হলে
# dirty_ratio: application block করে flush এই % dirty হলে

# Transparent Huge Pages — DISABLE করো (PostgreSQL এ harmful)
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
# /etc/rc.local বা systemd unit এ add করো persistent এর জন্য

# I/O Scheduler — SSD এর জন্য
cat /sys/block/sda/queue/scheduler  # current scheduler দেখো
echo 'none' > /sys/block/sda/queue/scheduler  # SSD: none (mq-deadline)
echo 'deadline' > /sys/block/sdb/queue/scheduler  # HDD: deadline

# File system — XFS recommended
# mkfs.xfs -f /dev/sdb1
# mount -o noatime,nodiratime /dev/sdb1 /var/lib/pgsql
# noatime: access time update করে না → I/O কমে

# ulimits — open files, processes বাড়াও
# /etc/security/limits.conf:
# postgres soft nofile 65536
# postgres hard nofile 65536
# postgres soft nproc  65536
# postgres hard nproc  65536
```

### CPU Optimization

```ini
# Parallel workers
max_worker_processes          = 16   # OS CPU core count
max_parallel_workers          = 8    # Parallel query workers (total)
max_parallel_workers_per_gather = 4  # Per query workers
parallel_tuple_cost           = 0.1  # Lower → more parallel
parallel_setup_cost           = 1000 # Overhead of starting parallel

# JIT Compilation (PostgreSQL 11+)
jit                   = on    # Enable JIT
jit_above_cost        = 100000  # Query cost > 100K → JIT compile
jit_inline_above_cost = 500000  # Inline functions
jit_optimize_above_cost = 500000

# JIT কখন useful:
# Complex analytics queries (OLAP)
# Heavy computation, many aggregates
# OLTP এ JIT overhead হতে পারে
# SET jit = off; -- per session এ disable করো OLTP এ যদি দরকার হয়
```

---

## 6.18 pgBench — Load Testing

pgBench হলো PostgreSQL এর built-in benchmarking tool। Performance testing এবং configuration tuning এ ব্যবহার করো।

### Setup এবং Basic Run

```bash
# Test database তৈরি করো
createdb -U postgres pgbench_test

# Initialize (test data তৈরি করো)
pgbench -U postgres -i -s 100 pgbench_test
# -i: initialize
# -s: scale factor (100 = ~10GB data)
# Tables: pgbench_accounts (10M rows), pgbench_branches, pgbench_tellers, pgbench_history

# Basic benchmark
pgbench -U postgres \
    -c 10 \     # 10 concurrent clients
    -j 2 \      # 2 threads
    -T 60 \     # 60 seconds duration
    pgbench_test
# অথবা number of transactions:
pgbench -U postgres -c 10 -j 2 -t 10000 pgbench_test

# Output:
# transaction type: <builtin: TPC-B (sort of)>
# scaling factor: 100
# number of clients: 10
# number of threads: 2
# duration: 60 s
# number of transactions actually processed: 58234
# latency average = 10.302 ms
# latency stddev = 5.231 ms
# tps = 970.553 (including connections establishing)
# tps = 970.665 (excluding connections establishing)
```

### Custom Scripts

```bash
# Custom query benchmark
cat > /tmp/select_test.sql << 'EOF'
\set user_id random(1, 1000000)
SELECT id, name, email FROM users WHERE id = :user_id;
EOF

pgbench -U postgres \
    -c 50 -j 4 -T 120 \
    -f /tmp/select_test.sql \
    mydb

# Mixed read/write benchmark
cat > /tmp/mixed.sql << 'EOF'
\set user_id random(1, 1000000)
\set amount random(1, 1000)

BEGIN;
SELECT balance FROM accounts WHERE user_id = :user_id FOR UPDATE;
UPDATE accounts SET balance = balance + :amount WHERE user_id = :user_id;
INSERT INTO transactions (user_id, amount, created_at) VALUES (:user_id, :amount, NOW());
COMMIT;
EOF

pgbench -U postgres -c 20 -j 4 -T 60 -f /tmp/mixed.sql mydb
```

### Configuration Compare করো

```bash
# Baseline: default settings
pgbench -U postgres -c 10 -T 60 mydb > /tmp/baseline.txt

# Tune করো (work_mem বাড়াও)
psql -c "ALTER SYSTEM SET work_mem = '64MB';"
psql -c "SELECT pg_reload_conf();"

# After tuning
pgbench -U postgres -c 10 -T 60 mydb > /tmp/tuned.txt

# Compare
grep tps /tmp/baseline.txt
grep tps /tmp/tuned.txt

# Latency percentiles দেখতে
pgbench -U postgres -c 50 -T 60 --latency-limit=100 mydb
# --latency-limit: এর বেশি latency = "late"
```

### pgBench Metrics Interpretation

```
TPS (Transactions per Second):
  970 TPS → প্রতি সেকেন্ডে 970 complete transactions
  Higher = Better

Latency Average:
  10.3 ms → Average transaction time
  Lower = Better

Latency Stddev:
  5.2 ms → Variance
  Low stddev = Consistent performance

Initial connection time:
  "including connections" vs "excluding connections"
  পার্থক্য বেশি হলে → connection setup overhead বেশি → pgBouncer দরকার
```

---

## 6.19 Query Rewrite Techniques

### EXISTS vs IN vs JOIN

```sql
-- Large IN list → NOT efficient
SELECT * FROM users WHERE id IN (1, 2, 3, ..., 10000);
-- ✅ Temporary table বা array ব্যবহার করো
SELECT * FROM users WHERE id = ANY(ARRAY[1,2,3,...]);
-- অথবা:
CREATE TEMP TABLE user_ids (id INT);
COPY user_ids FROM '/tmp/ids.csv';
SELECT u.* FROM users u JOIN user_ids t ON u.id = t.id;

-- EXISTS vs IN (subquery)
-- প্রায় same performance, planner usually same plan করে
-- কিন্তু EXISTS semantically clearer

-- ❌ IN subquery (লাখো rows এ slow হতে পারে)
SELECT * FROM users WHERE id IN (SELECT user_id FROM orders WHERE total > 100);
-- ✅ EXISTS (early exit — একটা match পেলেই বেরিয়ে আসে)
SELECT * FROM users u WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.id AND o.total > 100
);
-- ✅ JOIN (usually fastest, planner has most flexibility)
SELECT DISTINCT u.* FROM users u
JOIN orders o ON u.id = o.user_id AND o.total > 100;
-- বা semi-join:
SELECT u.* FROM users u
WHERE u.id IN (SELECT DISTINCT user_id FROM orders WHERE total > 100);
```

### CASE WHEN Optimization

```sql
-- Multiple UPDATE with CASE (single scan)
-- ❌ Multiple queries
UPDATE orders SET status = 'confirmed' WHERE status = 'pending' AND total > 500;
UPDATE orders SET status = 'cancelled' WHERE status = 'pending' AND total <= 0;

-- ✅ Single query
UPDATE orders
SET status = CASE
    WHEN status = 'pending' AND total > 500 THEN 'confirmed'
    WHEN status = 'pending' AND total <= 0  THEN 'cancelled'
    ELSE status
END
WHERE status = 'pending';

-- CASE in SELECT (computed column)
SELECT id, total,
    CASE
        WHEN total >= 1000 THEN 'platinum'
        WHEN total >= 500  THEN 'gold'
        WHEN total >= 100  THEN 'silver'
        ELSE 'bronze'
    END AS tier
FROM orders;
```

### Lateral JOIN — Row-wise Subquery

```sql
-- প্রতিটা user এর জন্য last 3 orders
-- ❌ Correlated subquery (N+1)
SELECT u.id, (SELECT json_agg(o) FROM orders o WHERE o.user_id = u.id
    ORDER BY o.created_at DESC LIMIT 3) AS recent_orders
FROM users u;

-- ✅ LATERAL JOIN (একবারই চলে)
SELECT u.id, u.name, o.id AS order_id, o.total
FROM users u
CROSS JOIN LATERAL (
    SELECT id, total FROM orders
    WHERE user_id = u.id
    ORDER BY created_at DESC
    LIMIT 3
) o;

-- LATERAL এর সাথে index:
-- orders(user_id, created_at DESC) index থাকলে প্রতিটা user এর জন্য index use করে
```

### VALUES এবং Unnest

```sql
-- Multiple row process করতে VALUES
SELECT * FROM (
    VALUES (1, 'Alice'), (2, 'Bob'), (3, 'Charlie')
) AS t(id, name)
JOIN users u ON u.id = t.id;

-- Array unnest দিয়ে batch process
SELECT unnest(ARRAY[1,2,3,4,5]) AS user_id;

-- Batch lookup
SELECT u.* FROM users u
JOIN unnest(ARRAY[1,2,3,4,5]) AS t(id) ON u.id = t.id;
```

### Date/Time Query Optimization

```sql
-- ❌ Function on indexed column → index নষ্ট হয়
WHERE DATE(created_at) = '2024-03-08'
WHERE EXTRACT(YEAR FROM created_at) = 2024

-- ✅ Range query
WHERE created_at >= '2024-03-08' AND created_at < '2024-03-09'
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'

-- ✅ BETWEEN (inclusive on both sides)
WHERE created_at BETWEEN '2024-03-08 00:00:00' AND '2024-03-08 23:59:59.999'
-- TIMESTAMPTZ এ সাবধান: timezone নিয়মিত convert হয়
WHERE created_at >= '2024-03-08 00:00:00+06' AND created_at < '2024-03-09 00:00:00+06'

-- NOW() vs CURRENT_TIMESTAMP vs CURRENT_DATE
SELECT NOW();                -- 2024-03-08 14:30:05.123+06 (TIMESTAMPTZ)
SELECT CURRENT_TIMESTAMP;   -- same as NOW()
SELECT CURRENT_DATE;        -- 2024-03-08 (DATE)
SELECT CURRENT_TIME;        -- 14:30:05.123+06 (TIMETZ)

-- Index দিয়ে efficient date range
CREATE INDEX idx_orders_created ON orders(created_at);
-- এখন range query fast: WHERE created_at >= '2024-03-01' AND created_at < '2024-04-01'
```

### Text Search Optimization

```sql
-- LIKE patterns:
-- 'prefix%'  → B-Tree index কাজ করে ✅
-- '%suffix'  → B-Tree কাজ করে না ❌ → pg_trgm
-- '%middle%' → B-Tree কাজ করে না ❌ → pg_trgm

-- pg_trgm — trigram similarity search
CREATE EXTENSION pg_trgm;

-- GIN index with trigrams
CREATE INDEX idx_trgm_product ON products USING GIN(name gin_trgm_ops);
SELECT * FROM products WHERE name ILIKE '%phone%';       -- index use করবে
SELECT * FROM products WHERE name % 'phoone';            -- fuzzy match
SELECT similarity('phone', 'fphone') AS score;           -- 0.0 to 1.0

-- Full-text search (better for large text)
-- to_tsvector: text → searchable tokens
-- to_tsquery:  search query
-- @@: match operator

CREATE INDEX idx_fts_products ON products USING GIN(
    to_tsvector('english', name || ' ' || COALESCE(description, ''))
);

SELECT name, ts_rank(
    to_tsvector('english', name || ' ' || description),
    to_tsquery('english', 'smartphone & android')
) AS rank
FROM products
WHERE to_tsvector('english', name || ' ' || description)
    @@ to_tsquery('english', 'smartphone & android')
ORDER BY rank DESC;

-- Generated column দিয়ে (PG 12+)
ALTER TABLE products ADD COLUMN search_vector TSVECTOR
    GENERATED ALWAYS AS (
        to_tsvector('english', name || ' ' || COALESCE(description, ''))
    ) STORED;
CREATE INDEX idx_fts ON products USING GIN(search_vector);
-- এখন to_tsvector() আর call করতে হবে না query তে:
SELECT * FROM products WHERE search_vector @@ to_tsquery('english', 'smartphone');
```

---

## 6.20 Performance Troubleshooting Workflow

Real production এ slow query পেলে systematic approach দরকার।

### Step 1 — Slow Query Identify করো

```sql
-- pg_stat_statements (বা pg_stat_monitor) থেকে top slow queries
SELECT
    LEFT(query, 100)                           AS query,
    calls,
    ROUND(total_exec_time::numeric, 2)         AS total_ms,
    ROUND(mean_exec_time::numeric, 2)          AS avg_ms,
    ROUND(stddev_exec_time::numeric, 2)        AS stddev_ms,
    rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Log থেকে (log_min_duration_statement এ caught হলে)
sudo grep "duration:" /var/lib/pgsql/16/data/log/postgresql-*.log | \
    awk '{print $NF, $0}' | sort -rn | head -20

-- Real-time slow queries
SELECT pid, now() - query_start AS duration, LEFT(query, 100)
FROM pg_stat_activity
WHERE state = 'active' AND now() - query_start > INTERVAL '5 seconds'
ORDER BY duration DESC;
```

### Step 2 — EXPLAIN করো

```sql
-- Problem query identify হলে EXPLAIN করো
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
<your slow query here>;

-- কী দেখবো:
-- Seq Scan on large table → index নেই বা index use হচ্ছে না
-- Hash Batches: N (N>1) → work_mem কম
-- Sort Method: external merge → work_mem কম (sort disk এ হচ্ছে)
-- rows=1000 (actual rows=100000) → Statistics wrong, ANALYZE দরকার
-- Nested Loop on large tables → Wrong join plan, statistics check
-- Filter: ... (rows removed by filter: N) → Index condition নয়, post-filter → partial index
```

### Step 3 — Root Cause Analysis

```
Seq Scan দেখলে:
  ① Index নেই → CREATE INDEX
  ② Index আছে কিন্তু use হচ্ছে না?
      → Function on column? → Expression Index
      → Leading wildcard LIKE? → pg_trgm
      → Implicit cast? → Fix data type
      → Planner wrong estimate? → ANALYZE, বেশি statistics
      → Table ছোট (1000 rows)? → Planner intentionally seq scan (সঠিক!)

High Buffer Read (disk I/O) দেখলে:
  → shared_buffers বাড়াও
  → effective_cache_size correct দাও
  → Working set RAM তে fit করছে না → RAM বাড়াও

Work_mem issue (Sort on disk, Hash batches > 1):
  → SET work_mem = '256MB'; EXPLAIN আবার চালাও
  → Better হলে → global বা session level work_mem বাড়াও

Statistics issue (estimate ≠ actual rows):
  → ANALYZE table;
  → ALTER TABLE t ALTER COLUMN c SET STATISTICS 500; ANALYZE;

Bloat issue (table size অনেক বেশি):
  → pg_repack চালাও
  → Autovacuum aggressive করো

Lock wait (query blocking এর কারণে slow):
  → pg_blocking_pids(pid) দেখো
  → Blocking query identify করো
  → Application logic fix করো (shorter transactions)
```

### Step 4 — Fix করো এবং Verify করো

```sql
-- Fix করার পর:
-- ① statistics reset করো
SELECT pg_stat_statements_reset();

-- ② Fix deploy করো (index, query change, config)

-- ③ কিছুক্ষণ পরে আবার check করো
SELECT mean_exec_time FROM pg_stat_statements WHERE query LIKE '%your_query%';

-- ④ EXPLAIN আবার করো এবং compare করো
-- আগে: Seq Scan cost=0..50000
-- পরে: Index Scan cost=0..8

-- ⑤ pgBench দিয়ে load test করো (regression নেই তো?)
pgbench -U postgres -c 10 -T 30 mydb

-- Production monitoring এ alert set করো
-- avg_ms > threshold হলে alert (PMM বা custom script)
```

### Quick Reference — Common Performance Issues

| Symptom | Likely Cause | Fix |
|---|---|---|
| Seq Scan on large table | Index নেই | CREATE INDEX |
| Index আছে তবু Seq Scan | Function on column, wrong type | Expression Index, fix type |
| LIKE '%word%' slow | Leading wildcard | pg_trgm GIN index |
| High disk reads | Buffer hit rate কম | shared_buffers বাড়াও |
| Sort on disk | work_mem কম | work_mem বাড়াও |
| Hash Batches > 1 | work_mem কম | work_mem বাড়াও |
| N+1 queries | ORM lazy loading | Eager load, JOIN |
| Large OFFSET slow | Offset scan | Keyset pagination |
| High idle in transaction | Long open transactions | idle_in_transaction_timeout |
| Table bloat | Dead tuples জমা | pg_repack, autovacuum tune |
| XID wraparound risk | VACUUM না চলা | VACUUM FREEZE |
| Deadlock | Lock order inconsistent | Consistent lock order |
| High connection count | No pooler | pgBouncer যোগ করো |
| Replica lag | High write load | sync_commit tune, hardware |
| Slow aggregate | HashAggregate batch | work_mem বাড়াও |
| Function call slow | Missing statistics | ANALYZE, stats target |

---

# 7. Backup & Recovery

## 7.0 Backup Types — সব কিছুর ভিত্তি

Backup strategy ঠিক করার আগে types গুলো বুঝতে হবে।

### Logical vs Physical Backup

```
Logical Backup (pg_dump, pg_dumpall):
  কী backup হয়: SQL statements বা binary rows
  
  ✅ Cross-version restore (PG 14 backup → PG 16 restore)
  ✅ Cross-platform (Linux backup → Windows restore)
  ✅ Selective restore (specific table, specific schema)
  ✅ Human-readable (plain SQL format)
  ❌ PITR সম্ভব না
  ❌ Large database এ slow
  ❌ Roles/tablespaces আলাদা করে backup দরকার (pg_dumpall)
  
  কখন ব্যবহার করবো:
    Dev/test environment restore
    Database migration (version upgrade)
    Selective table restore
    Small database (<50GB)
    Cross-platform transfer

Physical Backup (pg_basebackup, pgBackRest, Barman):
  কী backup হয়: Data directory এর exact binary copy + WAL
  
  ✅ PITR — যেকোনো specific time এ restore
  ✅ Fast restore (file copy, no SQL replay)
  ✅ Streaming replication setup এর base
  ✅ Database চলতে থাকে backup এর সময়
  ❌ Same major version (PG 16 → PG 16 only)
  ❌ Selective table restore নেই
  ❌ বড় size (সব data pages)
  
  কখন ব্যবহার করবো:
    Production disaster recovery
    HA standby setup
    PITR দরকার হলে
    Large database
```

### Full vs Differential vs Incremental

```
Full Backup:
  কী: পুরো database এর সম্পূর্ণ copy
  Size: Database size এর 100%
  Restore: শুধু এই একটা file → simple
  কখন: Weekly বা monthly base

  [Monday Full: 50GB]──────────────────────────────────
  
Differential Backup:
  কী: Last FULL backup এর পর থেকে changed data
  Size: Full এর পর কতটা change → বাড়তে থাকে
  Restore: Full + Latest Differential (2 files)
  কখন: Daily

  [Monday Full: 50GB]
         ├── [Tuesday Diff: 1GB  (Mon পর change)]
         ├── [Wednesday Diff: 2GB (Mon পর change — বাড়ছে)]
         └── [Thursday Diff: 3GB]
  
  Restore to Wednesday: Full(50GB) + WedDiff(2GB) = done

Incremental Backup:
  কী: Last backup (যেকোনো type) এর পর থেকে changed data
  Size: সবচেয়ে ছোট (শুধু নতুন changes)
  Restore: Full + সব Incremental order এ (complex)
  কখন: Daily বা hourly

  [Monday Full: 50GB]
         ├── [Tuesday Incr: 1GB  (Mon পর change)]
         │         └── [Wednesday Incr: 1GB (Tue পর change)]
         │                    └── [Thursday Incr: 1GB]
  
  Restore to Wednesday: Full + TueIncr + WedIncr = done
```

**Comparison Table:**
| | Full | Differential | Incremental |
|---|---|---|---|
| Backup size | 100% DB size | Medium, grows | Smallest (~daily change) |
| Backup time | Slowest | Medium | Fastest |
| Restore time | Fastest | Medium | Slowest |
| Restore complexity | Simple | Simple | Complex (chain) |
| Storage needed | Most | Medium | Least |
| pgBackRest option | `--type=full` | `--type=diff` | `--type=incr` |

**Production Recommended Strategy:**
```
Small DB (<100GB):
  Daily Full + Continuous WAL Archiving
  Retention: 7 days
  
Medium DB (100GB-1TB):
  Weekly Full (Sunday) + Daily Differential (Mon-Sat) + Continuous WAL
  Retention: 2 full + 14 diff
  
Large DB (>1TB):
  Monthly Full + Weekly Differential + Daily Incremental + Continuous WAL
  Retention: 1 full + 4 diff + 30 incr
```

**Hands-on: Backup Type Planning করো তোমার Database এর জন্য**

```bash
# Step 1: Database size দেখো — strategy choose করতে
psql -U postgres -c "
SELECT datname,
    pg_size_pretty(pg_database_size(datname)) AS size,
    pg_database_size(datname) AS size_bytes
FROM pg_database
WHERE datname NOT IN ('template0','template1')
ORDER BY size_bytes DESC;"

# Step 2: Daily write volume দেখো — incremental কতটা হবে estimate করো
psql -U postgres -d mydb -c "
SELECT
    tup_inserted AS daily_inserts,
    tup_updated  AS daily_updates,
    tup_deleted  AS daily_deletes,
    tup_inserted + tup_updated + tup_deleted AS total_writes
FROM pg_stat_database
WHERE datname = 'mydb';"

# Step 3: WAL generation rate দেখো
psql -U postgres -c "
SELECT pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), '0/0')) AS total_wal_generated;"
# অথবা hourly rate দেখো:
psql -U postgres -c "SELECT pg_current_wal_lsn();"  -- note this LSN
sleep 3600
psql -U postgres -c "SELECT pg_current_wal_lsn();"  -- compare

# Step 4: pgBackRest দিয়ে backup size estimate করো
sudo -u postgres pgbackrest --stanza=main info
# delta size দেখো — এটাই incremental backup এর আনুমানিক size

# Step 5: Backup করো এবং verify করো
sudo -u postgres pgbackrest --stanza=main --type=full backup
sudo -u postgres pgbackrest --stanza=main info
# Output তে backup size, duration দেখাবে
```

---

## 7.1 pg_dump — Single Database Logical Backup

```bash
# Plain SQL (portable, human-readable)
pg_dump -U postgres mydb > mydb_backup.sql

# Custom format (compressed, selective restore possible) ← Production recommended
pg_dump -U postgres -Fc mydb > mydb_backup.dump

# Tar format
pg_dump -U postgres -Ft mydb > mydb_backup.tar

# Directory format (parallel possible)
pg_dump -U postgres -Fd -f /backup/mydb_dir mydb

# Parallel dump (directory format only, -j = jobs)
pg_dump -U postgres -Fd -j 4 -f /backup/mydb_dir mydb

# Schema only
pg_dump -U postgres -s mydb > mydb_schema.sql

# Data only
pg_dump -U postgres -a mydb > mydb_data.sql

# Specific table
pg_dump -U postgres -t users mydb > users_backup.sql
pg_dump -U postgres -t 'public.*' mydb > all_public_tables.sql

# Exclude table
pg_dump -U postgres -T audit_logs mydb > mydb_no_logs.sql

# Compressed pipe
pg_dump -U postgres mydb | gzip > mydb_$(date +%Y%m%d).sql.gz

# Remote server
PGPASSWORD=secret pg_dump -U postgres -h 172.16.93.140 mydb | gzip > remote_backup.sql.gz

# With verbose progress
pg_dump -U postgres -Fc --verbose mydb -f mydb.dump
```

**Format comparison:**
| Format | Option | Compressed | Parallel Restore | Selective Restore |
|---|---|---|---|---|
| Plain SQL | `-Fp` | No | No | No |
| Custom | `-Fc` | Yes (built-in) | No | Yes |
| Tar | `-Ft` | No | No | Yes |
| Directory | `-Fd` | Yes | Yes | Yes |

---

## 7.2 pg_dumpall — Cluster-wide Backup

Roles, tablespaces এবং সব database একসাথে backup।

```bash
# Full cluster backup (roles + tablespaces + all databases)
pg_dumpall -U postgres > cluster_full_backup.sql

# Globals only (roles, tablespaces — data নেই)
# নতুন server এ migrate করার আগে এটা দিয়ে roles create করো
pg_dumpall -U postgres --globals-only > globals_backup.sql

# Schema only (structure, no data)
pg_dumpall -U postgres --schema-only > schema_backup.sql

# Compressed
pg_dumpall -U postgres | gzip > cluster_backup_$(date +%Y%m%d_%H%M%S).sql.gz
```

---

## 7.3 pg_restore — Restore Backup

```bash
# Custom/Directory/Tar format restore
pg_restore -U postgres -d mydb mydb_backup.dump

# Database তৈরি করে restore (database আগে থেকে নেই)
pg_restore -U postgres -C -d postgres mydb_backup.dump
# -C: CREATE DATABASE করবে, তারপর নতুন DB তে restore

# Specific table restore
pg_restore -U postgres -d mydb -t users mydb_backup.dump
pg_restore -U postgres -d mydb -t orders mydb_backup.dump

# Schema only
pg_restore -U postgres -d mydb -s mydb_backup.dump

# Data only
pg_restore -U postgres -d mydb -a mydb_backup.dump

# Parallel restore (directory format)
pg_restore -U postgres -d mydb -j 4 mydb_backup.dump

# Verbose
pg_restore -U postgres -d mydb --verbose mydb_backup.dump 2>&1 | tee restore.log

# Archive এ কী আছে দেখো
pg_restore -l mydb_backup.dump

# Plain SQL restore
psql -U postgres -d mydb < mydb_backup.sql
# অথবা compressed:
gunzip -c mydb_backup.sql.gz | psql -U postgres -d mydb
```

---

## 7.4 pg_basebackup — Physical Backup

Full cluster physical copy। PITR এর base হিসেবে ব্যবহার হয়।

```bash
# Replication user তৈরি করো
# (pg_hba.conf এ replication connection allow করতে হবে)
CREATE USER replicator REPLICATION LOGIN PASSWORD 'ReplicaPass@2024!';

# pg_hba.conf এ যোগ করো:
# host replication replicator 172.16.93.0/24 scram-sha-256

# Basic backup
pg_basebackup -U replicator -h localhost -D /backup/base -P

# WAL stream করে নাও (recommended)
pg_basebackup -U replicator -h localhost \
    -D /backup/base \
    -Xs \         # WAL stream (-Xs = WAL method stream)
    -P \          # Progress দেখাও
    --checkpoint=fast  # Fast checkpoint trigger

# Compressed tar
pg_basebackup -U replicator -h localhost \
    -D /backup/base \
    -Ft -z \      # Tar format, gzip compress
    -Xs -P

# Standby setup এর জন্য (automatic recovery config তৈরি করে)
pg_basebackup -U replicator -h 172.16.93.140 \
    -D /var/lib/pgsql/16/data \
    -Xs -R -P
# -R: standby.signal তৈরি করে + primary_conninfo postgresql.auto.conf এ লেখে

# Verify backup
pg_basebackup -U replicator -h localhost --verify -D /backup/base
```

---

## 7.5 WAL Archiving

PITR এর জন্য WAL segments archive করা। Base backup + WAL archive = যেকোনো সময়ে restore।

```ini
# postgresql.conf
wal_level      = replica
archive_mode   = on
archive_command = 'test ! -f /archive/wal/%f && cp %p /archive/wal/%f'
# %p = WAL file এর full path
# %f = filename only
# test ! -f: file না থাকলেই copy (re-archive prevent)

archive_timeout = 300  # 5 মিনিটে একবার force archive (idle server এও)
```

```bash
mkdir -p /archive/wal
chown postgres:postgres /archive/wal
chmod 700 /archive/wal

# Archive status দেখো
psql -c "SELECT * FROM pg_stat_archiver;"
# last_archived_wal: শেষ archive হওয়া WAL
# failed_count: 0 হওয়া উচিত

# Manual WAL switch করো (current WAL archive করো)
psql -c "SELECT pg_switch_wal();"
```

---

## 7.6 Point-in-Time Recovery (PITR)

**Scenario:** সকাল ১০টায় ভুলে `DELETE FROM orders WHERE 1=1` হয়েছে। রাত ২টায় backup আছে।

```bash
# Step 1: Server stop করো
sudo systemctl stop postgresql-16

# Step 2: Current broken data সরাও
mv /var/lib/pgsql/16/data /var/lib/pgsql/16/data.broken

# Step 3: Base backup থেকে restore করো
mkdir -p /var/lib/pgsql/16/data
# Custom format হলে:
pg_restore -U postgres -d postgres /backup/base/base.dump
# Tar হলে:
tar -xzf /backup/base/base.tar.gz -C /var/lib/pgsql/16/data/
chown -R postgres:postgres /var/lib/pgsql/16/data

# Step 4: Recovery configuration যোগ করো
cat >> /var/lib/pgsql/16/data/postgresql.conf << 'EOF'
restore_command = 'cp /archive/wal/%f %p'
recovery_target_time   = '2024-03-08 09:59:59+06'  # DELETE এর ঠিক আগে
recovery_target_action = 'promote'   # Recovery হলে Primary হয়ে যাবে
EOF

# ─── Recovery Target Options ─── (একটাই ব্যবহার করো)

# Option 1: Time-based (সবচেয়ে common)
# recovery_target_time = '2024-03-08 09:59:59+06'

# Option 2: LSN-based (সবচেয়ে precise)
# recovery_target_lsn = '0/3A1F2B8'
# কীভাবে LSN জানবো?
# Primary এ: SELECT pg_current_wal_lsn();  -- disaster এর আগের LSN note করো
# WAL log এ: pg_waldump /archive/wal/000... | grep "your marker"

# Option 3: Transaction ID
# recovery_target_xid = '12345'

# Option 4: Named restore point (সবচেয়ে reliable — production maintenance এ)
# Production এ major operation এর আগে restore point তৈরি করো:
# SELECT pg_create_restore_point('before_schema_migration_v2');
# তারপর: recovery_target_name = 'before_schema_migration_v2'

# Option 5: Immediate (WAL consistency হলেই stop)
# recovery_target = 'immediate'  -- earliest consistent point

# recovery_target_inclusive:
# true  (default): target এর পরের transaction পর্যন্ত (target সহ)
# false: target এর ঠিক আগে stop (target বাদ)
# recovery_target_inclusive = true

# recovery_target_action:
# 'promote'  (default PG 12+): Recovery হলে Primary হয়ে যাবে
# 'pause':   Recovery pause হবে, manually resume দরকার
# 'shutdown': Recovery complete হলে server বন্ধ হবে (backup verify এ useful)

# Step 5: Recovery signal তৈরি করো (PostgreSQL 12+)
touch /var/lib/pgsql/16/data/recovery.signal
# এই file থাকলে PostgreSQL recovery mode এ start হয়

# Step 6: Start করো
sudo systemctl start postgresql-16

# Log দেখো — recovery progress
sudo tail -f /var/lib/pgsql/16/data/log/postgresql-*.log
# দেখবে: "recovery stopping before commit of transaction..."
# তারপর: "database system is ready to accept connections"

# Step 7: Verify
psql -c "SELECT COUNT(*) FROM orders;"  -- সব data আছে?
psql -c "SELECT pg_is_in_recovery();"   -- f = Primary (recovery complete)
```


### PITR Targets — চারটা উপায়ে exact point এ restore করা যায়

```sql
-- ─── Target Type 1: Time (সবচেয়ে common) ───
-- recovery_target_time = '2024-03-08 09:59:59+06'
-- "ঠিক এই সময়ের আগের অবস্থায় যাও"

-- ─── Target Type 2: LSN (সবচেয়ে precise) ───
-- recovery_target_lsn = '0/3A1F2B8'
-- Specific WAL position এ restore
-- কখন দরকার: Log এ exact LSN জানা থাকলে (e.g., error এর আগের LSN)

-- Current LSN দেখো (disaster এর আগে note করো)
SELECT pg_current_wal_lsn() AS current_lsn,
    pg_walfile_name(pg_current_wal_lsn()) AS wal_file;

-- ─── Target Type 3: Transaction ID ───
-- recovery_target_xid = '12345'
-- Specific transaction এর পরে stop করো

-- ─── Target Type 4: Named Restore Point (সবচেয়ে safe) ───
-- Major operation এর আগে restore point তৈরি করো
SELECT pg_create_restore_point('before_batch_delete');
-- Returns: restore point LSN

-- তারপর:
-- recovery_target_name = 'before_batch_delete'
-- Exactly এই point এ restore হবে
```

```bash
# ─── Named Restore Point Workflow ───
# Production এ risky operation এর আগে সবসময় এটা করো

# Step 1: Operation এর আগে restore point তৈরি করো
psql -U postgres -c "SELECT pg_create_restore_point('before_migration_20240308');"
# '0/3A1F2B8' ← এই LSN save করো

# Step 2: Operation করো
psql -U postgres -d mydb -f /tmp/migration.sql

# Step 3: Problem হলে restore করো
sudo systemctl stop postgresql-16
sudo -u postgres pgbackrest --stanza=main restore \
    --type=name \
    --target=before_migration_20240308 \
    --target-action=promote
sudo systemctl start postgresql-16

# ─── LSN দিয়ে PITR ───
# WAL file এ কোন LSN এ কী হয়েছে দেখো
pg_waldump /var/lib/pgsql/16/data/pg_wal/000000010000000000000003 | \
    grep -A2 "DELETE" | head -20

# Specific LSN এ restore করো
sudo -u postgres pgbackrest --stanza=main restore \
    --type=lsn \
    --target='0/3A1F2B8' \
    --target-action=promote

# recovery_target_action values:
# promote  → recovery হলে Primary হয়ে যাবে (default for most)
# pause    → recovery থামবে, manually resume করতে হবে
# shutdown → recovery হলে server shutdown হবে
```

```sql
-- ─── Recovery target inclusive vs exclusive ───
-- recovery_target_inclusive = true (default)
--   target transaction/time সহ stop করবে
-- recovery_target_inclusive = false
--   target transaction/time এর ঠিক আগে stop করবে
-- "DELETE এর transaction টা undo করতে চাই"
--   → target = DELETE transaction ID
--   → inclusive = false

-- ─── PITR কতটুকু পর্যন্ত possible দেখো ───
-- WAL archive minimum LSN
SELECT min_archive_point FROM pg_stat_archiver;
-- এর আগে PITR সম্ভব না (WAL নেই)

-- pgBackRest দিয়ে কোন সময় পর্যন্ত restore possible
-- sudo -u postgres pgbackrest --stanza=main info
-- archive: 20240301-000000 to 20240308-142359
-- মানে: 2024-03-01 00:00 থেকে 2024-03-08 14:23 পর্যন্ত যেকোনো সময়ে
```

---

## 7.7 pgBackRest — Production Backup Solution

Production এ pg_dump এবং manual WAL archive যথেষ্ট না। pgBackRest দিয়ে automated, reliable backup।

**pgBackRest vs manual:**
```
Manual (pg_dump + WAL archive):
  - Restore complex, manual steps
  - Parallel restore নেই
  - Encryption নেই by default
  - Multiple server manage কঠিন

pgBackRest:
  + Full/Differential/Incremental backup
  + Parallel backup এবং restore
  + Built-in compression (lz4, zstd)
  + Encryption
  + Remote backup (S3, Azure, GCS)
  + Automatic retention management
  + Simple PITR
  + Verify backup integrity
```

```bash
# Install
sudo dnf install -y pgbackrest

# Configuration
sudo mkdir -p /etc/pgbackrest
sudo nano /etc/pgbackrest/pgbackrest.conf
```

```ini
[global]
repo1-path             = /var/lib/pgbackrest
repo1-retention-full   = 2          # 2টা full backup রাখো
repo1-retention-diff   = 14         # 14টা differential
repo1-cipher-type      = aes-256-cbc  # Encryption
repo1-cipher-pass      = YourEncryptionKey2024!
log-level-console      = info
log-level-file         = debug
log-path               = /var/log/pgbackrest
start-fast             = y          # Fast checkpoint trigger

[main]                              # Stanza name (যেকোনো নাম দাও)
pg1-path               = /var/lib/pgsql/16/data
pg1-port               = 5432
pg1-user               = postgres
```

```bash
# Archive location তৈরি করো
sudo mkdir -p /var/lib/pgbackrest
sudo chown -R postgres:postgres /var/lib/pgbackrest

# postgresql.conf এ archive_command আপডেট করো
# archive_command = 'pgbackrest --stanza=main archive-push %p'
# archive_mode = on
# wal_level = replica

sudo systemctl reload postgresql-16

# Stanza তৈরি করো (একবারই)
sudo -u postgres pgbackrest --stanza=main stanza-create

# Verify setup
sudo -u postgres pgbackrest --stanza=main check

# Full backup
sudo -u postgres pgbackrest --stanza=main --type=full backup

# Differential backup (last full থেকে changed data)
sudo -u postgres pgbackrest --stanza=main --type=diff backup

# Incremental backup (last backup থেকে changed data)
sudo -u postgres pgbackrest --stanza=main --type=incr backup

# Backup list দেখো
sudo -u postgres pgbackrest --stanza=main info

# Restore — Full (latest)
sudo systemctl stop postgresql-16
sudo -u postgres pgbackrest --stanza=main restore

# PITR restore
sudo -u postgres pgbackrest --stanza=main restore \
    --target="2024-03-08 09:59:59+06" \
    --target-action=promote \
    --type=time

sudo systemctl start postgresql-16

# S3 এ backup (remote)
```

```ini
# S3 config pgbackrest.conf এ
[global]
repo1-type         = s3
repo1-s3-bucket    = my-pg-backup-bucket
repo1-s3-endpoint  = s3.amazonaws.com
repo1-s3-region    = ap-southeast-1
repo1-s3-key       = AKIAIOSFODNN7EXAMPLE
repo1-s3-key-secret = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

```bash
# Cron automation
sudo crontab -u postgres -e
# Full backup: রবিবার রাত ১টায়
0 1 * * 0 pgbackrest --stanza=main --type=full backup >> /var/log/pgbackrest/cron.log 2>&1
# Differential: সোম-শনি রাত ১টায়
0 1 * * 1-6 pgbackrest --stanza=main --type=diff backup >> /var/log/pgbackrest/cron.log 2>&1
```

---

## 7.8 Barman — Multi-Server Backup Management

Centralized backup management। একটা backup server থেকে multiple PostgreSQL server এর backup।

```bash
# Backup server এ install করো
sudo dnf install -y barman barman-cli

# /etc/barman.conf (global config)
```

```ini
[barman]
barman_home    = /var/lib/barman
barman_user    = barman
log_file       = /var/log/barman/barman.log
compression    = gzip
reuse_backup   = link
backup_method  = rsync
```

```ini
# /etc/barman.d/pgprimary.conf (per server)
[pgprimary]
description        = "Primary PostgreSQL Server"
conninfo           = host=172.16.93.140 user=barman dbname=postgres
ssh_command        = ssh postgres@172.16.93.140
backup_method      = rsync
archiver           = on
streaming_archiver = on
streaming_conninfo = host=172.16.93.140 user=barman_streaming
slot_name          = barman_slot
retention_policy   = RECOVERY WINDOW OF 7 DAYS
minimum_redundancy = 1
```

```sql
-- PostgreSQL server এ users তৈরি করো
CREATE USER barman SUPERUSER PASSWORD 'BarmanPass@2024!';
CREATE USER barman_streaming REPLICATION PASSWORD 'StreamPass@2024!';

-- pg_hba.conf এ:
-- host all         barman          172.16.93.150/32 scram-sha-256
-- host replication barman_streaming 172.16.93.150/32 scram-sha-256
```

```bash
# SSH key setup করো (barman server থেকে PG server এ passwordless SSH)
sudo -u barman ssh-keygen -t rsa
sudo -u barman ssh-copy-id postgres@172.16.93.140

# Operations
barman check pgprimary             # Health check
barman switch-wal pgprimary        # WAL switch trigger
barman cron                        # Maintenance (cron এ চালাও)
barman backup pgprimary            # Backup নাও
barman list-backup pgprimary       # Backup list
barman show-backup pgprimary latest  # Latest backup details

# Restore
barman recover pgprimary latest /var/lib/pgsql/16/data \
    --target-time "2024-03-08 09:59:59" \
    --remote-ssh-command "ssh postgres@172.16.93.140"

# Cron এ
* * * * * barman cron              # প্রতি minute
0 2 * * * barman backup pgprimary  # রাত ২টায় backup
```

**pgBackRest vs Barman:**
| Feature | pgBackRest | Barman |
|---|---|---|
| Multi-server | Multiple stanzas | Yes, designed for it |
| S3/Cloud | ✅ Native | Limited |
| Parallel backup/restore | ✅ | Limited |
| Encryption | ✅ | External |
| Complexity | Medium | Medium-High |
| Production choice | Single/Few servers | Many servers, enterprise |

---

## 7.9 Backup Best Practices

```
3-2-1 Rule:
  ✅ 3 copies of data
  ✅ 2 different storage media (local disk + cloud/tape)
  ✅ 1 offsite copy (different datacenter বা cloud region)

Strategy:
  ✅ Physical backup + Continuous WAL archiving = PITR capability
  ✅ Weekly/Monthly Full + Daily Differential + WAL = production standard
  ✅ Logical backup আলাদা রাখো (cross-version restore এর জন্য)
  ✅ Backup encryption (pgBackRest built-in AES-256-CBC)
  ✅ Backup compression (zstd বা lz4 → fast + small)

কোনটা কখন:
  pg_dump:        Single DB, migration, selective restore, version upgrade
  pg_dumpall:     Full cluster migration, roles + all databases
  pg_basebackup:  Physical backup, Standby setup, PITR base
  pgBackRest:     Production: PITR + automation + encryption + compression + S3
  Barman:         Multiple server centralized management, DBA team

Performance:
  ✅ Replica থেকে backup নাও (Primary এ load পড়বে না)
  ✅ Off-peak hours এ (রাত ২-৩টা)
  ✅ Compression সবসময় on (lz4 for speed, zstd for size)
  ✅ Parallel backup/restore (-j workers)
  ✅ Delta/Incremental backup (block-level change tracking)

Testing:
  ✅ Monthly restore test on separate server (not on production!)
  ✅ Backup integrity verify করো (pgBackRest verify)
  ✅ RTO/RPO টেস্ট করো — "কতক্ষণে restore হয়?"
  ✅ Recovery runbook তৈরি করো এবং practice করো
```

**Hands-on: Backup System Setup করো এবং চালু হয়েছে কিনা Verify করো**

```bash
# ─── Step 1: WAL archiving চালু আছে কিনা check করো ───
psql -U postgres -c "SHOW archive_mode; SHOW archive_command;"
psql -U postgres -c "SELECT * FROM pg_stat_archiver;"
# last_archived_wal: সম্প্রতি archive হওয়া WAL
# failed_count: 0 হওয়া উচিত

# ─── Step 2: Archive directory তৈরি এবং permission ───
sudo mkdir -p /archive/wal
sudo chown postgres:postgres /archive/wal
sudo chmod 700 /archive/wal
ls -la /archive/wal/

# ─── Step 3: Manual WAL switch করে archive test করো ───
psql -U postgres -c "SELECT pg_switch_wal();"
sleep 5
ls -lh /archive/wal/ | tail -3
# নতুন WAL file দেখা গেলে archiving কাজ করছে ✅

# ─── Step 4: pgBackRest stanza setup (production backup) ───
sudo -u postgres pgbackrest --stanza=main stanza-create
sudo -u postgres pgbackrest --stanza=main check
# INFO: check command end: completed successfully ✅

# ─── Step 5: First full backup নাও ───
time sudo -u postgres pgbackrest --stanza=main --type=full backup
# Duration note করো — এটা তোমার full backup RTO এর একটা অংশ

# ─── Step 6: Backup info দেখো ───
sudo -u postgres pgbackrest --stanza=main info
# Output এ দেখবে:
#   stanza: main
#   status: ok
#   backup: 20240308-020000F (full, size: XX.XGB, compressed: XX.XGB)
#   WAL archive min/max: 000...001/000...010

# ─── Step 7: Automated backup cron setup করো ───
sudo crontab -u postgres -e
# যোগ করো:
# Full backup: প্রতি রবিবার রাত ২টায়
# 0 2 * * 0 pgbackrest --stanza=main --type=full backup >> /var/log/pgbackrest/cron.log 2>&1
# Differential: সোম-শনি রাত ২টায়
# 0 2 * * 1-6 pgbackrest --stanza=main --type=diff backup >> /var/log/pgbackrest/cron.log 2>&1

# Verify cron
sudo crontab -u postgres -l
```

```bash
# ─── Backup completeness quick check script ───
cat > /usr/local/bin/backup_status.sh << 'SCRIPT'
#!/bin/bash
echo "=== Backup Status Check - $(date) ==="

echo ""
echo "--- WAL Archive ---"
psql -U postgres -t -A -c "
SELECT 'Last archived: '||COALESCE(last_archived_wal,'none')
    ||' ('||COALESCE((now()-last_archived_time)::text,'never')||' ago)'
    ||' | Failures: '||failed_count
FROM pg_stat_archiver;"

echo ""
echo "--- pgBackRest Backups ---"
pgbackrest --stanza=main info 2>/dev/null | grep -E "status|backup:|WAL" || echo "pgBackRest not configured"

echo ""
echo "--- Archive Directory Size ---"
du -sh /archive/wal/ 2>/dev/null || echo "No archive dir"

echo ""
echo "--- Last 3 WAL Archive Files ---"
ls -lht /archive/wal/ 2>/dev/null | head -4
SCRIPT
chmod +x /usr/local/bin/backup_status.sh
sudo -u postgres /usr/local/bin/backup_status.sh
```


---

## 7.10 Backup Verification — নিশ্চিত করো Backup কাজ করবে

Backup আছে কিন্তু corrupt বা incomplete হলে বিপদের সময় কাজে আসবে না।

```bash
# pgBackRest verify — stanza এর সব backup check করো
sudo -u postgres pgbackrest --stanza=main verify

# Output:
# INFO: verify command begin
# INFO: archive: 16-1/000000010000000000000001 (1 segment)
# INFO: backup: 20240308-010000F (full backup) verified
# INFO: backup: 20240308-010000F_20240309-010000D (diff) verified
# WARN: archive gap: 000000010000000000000005..000000010000000000000007
# → Archive gap পেলে WAL segments হারিয়েছে → PITR affected!

# pg_dump verify করো
pg_dump -U postgres -Fc mydb > /backup/mydb.dump
pg_restore --list /backup/mydb.dump | head -20  # Valid কিনা দেখো
pg_restore -U postgres -d postgres -C /backup/mydb.dump  # Actually restore করে test করো

# pg_basebackup verify (PG 15+)
pg_basebackup -U replicator -h localhost --verify-checksums -D /tmp/verify_base
# Data checksums verify করে

# Checksum enable করা আছে কিনা দেখো
psql -c "SHOW data_checksums;"
# on → corruption detection active
# initdb --data-checksums দিয়ে enable করতে হয় (একবারই, restart লাগে না)
# existing database এ: pg_checksums --enable --pgdata=/var/lib/pgsql/16/data
# (server offline থাকতে হবে, সময় লাগে)

# Manual checksum verification
md5sum /backup/mydb.dump > /backup/mydb.dump.md5
# Restore এর আগে verify:
md5sum -c /backup/mydb.dump.md5
```

```sql
-- Backup পরে database এ verify করো
-- Row count match করে?
SELECT relname, n_live_tup
FROM pg_stat_user_tables
ORDER BY n_live_tup DESC
LIMIT 10;

-- Critical tables এর data sample check করো
SELECT COUNT(*) FROM orders;
SELECT MAX(created_at) FROM orders;  -- সর্বশেষ data আছে?
SELECT * FROM orders ORDER BY id DESC LIMIT 5;
```

---

## 7.11 Restore Testing — Backup এর সবচেয়ে গুরুত্বপূর্ণ Part

Backup এর চেয়েও গুরুত্বপূর্ণ হলো restore test। যে backup কখনো restore হয়নি সেটা backup নয়।

### Monthly Restore Drill

```bash
# ─────────────────────────────────────────────────────
# STEP 1: Restore environment prepare করো (spare server)
# ─────────────────────────────────────────────────────
# Production এর কাছাকাছি spec এর test server
# PostgreSQL same version install করো
sudo dnf install -y postgresql16-server postgresql16-contrib

# ─────────────────────────────────────────────────────
# STEP 2: Physical restore (pgBackRest দিয়ে)
# ─────────────────────────────────────────────────────
# pgbackrest.conf কপি করো test server এ (repo path adjust করো)
sudo -u postgres pgbackrest --stanza=main \
    --pg1-path=/var/lib/pgsql/16/data \
    restore

sudo systemctl start postgresql-16

# ─────────────────────────────────────────────────────
# STEP 3: Data integrity check করো
# ─────────────────────────────────────────────────────
psql -U postgres -c "SELECT COUNT(*) FROM orders;"
psql -U postgres -c "SELECT COUNT(*) FROM users;"
psql -U postgres -c "SELECT MAX(created_at) FROM orders;"
# Production এর row count এর সাথে মেলাও

# ─────────────────────────────────────────────────────
# STEP 4: PITR test করো
# ─────────────────────────────────────────────────────
# 1 ঘন্টা আগের data restore হচ্ছে?
sudo -u postgres pgbackrest --stanza=main \
    --type=time \
    "--target=2024-03-08 14:00:00+06" \
    --target-action=promote \
    restore
sudo systemctl start postgresql-16
# Verify করো: যে data এই time এর পরে insert হয়েছে সেটা নেই
psql -c "SELECT MAX(created_at) FROM orders;"  # 14:00 এর কাছাকাছি হওয়া উচিত

# ─────────────────────────────────────────────────────
# STEP 5: RTO পরিমাপ করো
# ─────────────────────────────────────────────────────
# শুরু থেকে শেষ পর্যন্ত কতক্ষণ লাগল? → এটাই তোমার real RTO
# Document করো এবং stakeholders কে জানাও
echo "Full restore time: X minutes" >> /var/log/backup_drill.log
echo "Data verified: $(date)" >> /var/log/backup_drill.log
```

### Logical Backup Restore Test

```bash
# Test database তৈরি করো
createdb -U postgres mydb_restore_test

# Restore করো
pg_restore -U postgres -d mydb_restore_test /backup/mydb.dump
# অথবা plain SQL:
psql -U postgres -d mydb_restore_test < /backup/mydb.sql.gz

# Row count compare করো
psql -U postgres -d mydb -c "SELECT relname, n_live_tup FROM pg_stat_user_tables ORDER BY relname;" > /tmp/prod_counts.txt
psql -U postgres -d mydb_restore_test -c "SELECT relname, n_live_tup FROM pg_stat_user_tables ORDER BY relname;" > /tmp/restore_counts.txt
diff /tmp/prod_counts.txt /tmp/restore_counts.txt
# Difference থাকলে investigate করো

# Cleanup
dropdb -U postgres mydb_restore_test
```

---

## 7.12 Backup Encryption

Production এ backup অবশ্যই encrypt করা উচিত — বিশেষত cloud বা offsite storage তে।

```ini
# pgBackRest encryption (AES-256-CBC)
# pgbackrest.conf এ:
[global]
repo1-cipher-type = aes-256-cbc
repo1-cipher-pass = YourVeryLongAndRandomEncryptionKey2024!

# Key management:
# ① Key টা backup server এ secure store করো
# ② Key হারালে backup decrypt করা যাবে না! → Key backup আলাদাভাবে
# ③ Password Manager বা HashiCorp Vault এ রাখো
# ④ Key rotation: নতুন key দিয়ে নতুন full backup নাও
```

```bash
# Encrypted backup তৈরি
sudo -u postgres pgbackrest --stanza=main --type=full backup
# Automatically encrypted — repo তে cipher_type file থাকবে

# Verify encryption
sudo -u postgres pgbackrest --stanza=main info
# cipher: aes-256-cbc দেখাবে

# pg_dump এ OpenSSL দিয়ে encrypt (manual)
pg_dump -U postgres -Fc mydb | \
    openssl enc -aes-256-cbc -pbkdf2 -pass pass:YourPassword \
    > /backup/mydb_encrypted.dump.enc

# Decrypt করে restore
openssl enc -d -aes-256-cbc -pbkdf2 -pass pass:YourPassword \
    -in /backup/mydb_encrypted.dump.enc | \
    pg_restore -U postgres -d mydb
```

---

## 7.13 Backup Monitoring এবং Alerting

Backup fail হলে জানতে হবে। Silent failure সবচেয়ে বিপদজনক।

```sql
-- WAL Archive status — সবচেয়ে গুরুত্বপূর্ণ backup metric
SELECT
    last_archived_wal,
    last_archived_time,
    last_failed_wal,
    last_failed_time,
    failed_count,
    now() - last_archived_time AS since_last_archive
FROM pg_stat_archiver;

-- Alert condition:
-- failed_count > 0 → Archive এ সমস্যা
-- since_last_archive > archive_timeout × 2 → Archive stuck
-- last_failed_time > last_archived_time → আবার fail হচ্ছে
```

```bash
# Backup monitoring script
cat > /usr/local/bin/backup_monitor.sh << 'SCRIPT'
#!/bin/bash
PSQL="psql -U postgres -t -A"
ALERT_EMAIL="dba@company.com"
MAX_ARCHIVE_AGE_MINUTES=30

# 1. Archive failure check
FAILED=$(${PSQL} -c "SELECT failed_count FROM pg_stat_archiver;")
if [ "$FAILED" -gt 0 ]; then
    LAST_FAIL=$(${PSQL} -c "SELECT last_failed_time FROM pg_stat_archiver;")
    echo "WARNING: WAL archive failures: $FAILED. Last fail: $LAST_FAIL" | \
        mail -s "ALERT: PostgreSQL Archive Failure" $ALERT_EMAIL
fi

# 2. Archive staleness check
ARCHIVE_AGE=$(${PSQL} -c "SELECT EXTRACT(EPOCH FROM (now() - last_archived_time))/60 FROM pg_stat_archiver;")
if (( $(echo "$ARCHIVE_AGE > $MAX_ARCHIVE_AGE_MINUTES" | bc -l) )); then
    echo "WARNING: Last archive was ${ARCHIVE_AGE} minutes ago" | \
        mail -s "ALERT: PostgreSQL Archive Stale" $ALERT_EMAIL
fi

# 3. pgBackRest last backup age check
LAST_BACKUP=$(pgbackrest --stanza=main --output=json info | \
    python3 -c "import sys,json; d=json.load(sys.stdin); \
    print(d[0]['backup'][-1]['timestamp']['stop'])" 2>/dev/null)
if [ -n "$LAST_BACKUP" ]; then
    BACKUP_AGE=$(( ($(date +%s) - $LAST_BACKUP) / 3600 ))
    if [ "$BACKUP_AGE" -gt 26 ]; then  # 26 hours (daily backup expected)
        echo "WARNING: Last backup was ${BACKUP_AGE} hours ago" | \
            mail -s "ALERT: PostgreSQL Backup Old" $ALERT_EMAIL
    fi
fi

echo "$(date) Backup check complete" >> /var/log/backup_monitor.log
SCRIPT

chmod +x /usr/local/bin/backup_monitor.sh
# Cron এ: প্রতি 30 মিনিটে
echo "*/30 * * * * postgres /usr/local/bin/backup_monitor.sh" | \
    sudo tee /etc/cron.d/pg_backup_monitor
```

```bash
# pgBackRest এ backup info দেখো (JSON output)
pgbackrest --stanza=main --output=json info

# Output:
# [
#   {
#     "backup": [
#       {
#         "archive": {"start": "...", "stop": "..."},
#         "backrest": {"version": "2.49"},
#         "database": {"version": "16"},
#         "info": {
#           "delta": 1073741824,    # bytes changed since last
#           "size": 53687091200,    # full backup size
#           "repository": {"delta": 104857600, "size": 5368709120}
#         },
#         "label": "20240308-010000F",
#         "timestamp": {"start": 1709852400, "stop": 1709852800},
#         "type": "full"
#       }
#     ],
#     "status": {"code": 0, "message": "ok"}
#   }
# ]
```

---

## 7.14 Backup Tool Comparison — কোনটা কখন

| Feature | pg_dump | pg_basebackup | pgBackRest | Barman |
|---|---|---|---|---|
| **Type** | Logical | Physical | Physical | Physical |
| **PITR** | ❌ | Limited | ✅ Full | ✅ Full |
| **Full backup** | ✅ | ✅ | ✅ | ✅ |
| **Differential** | ❌ | ❌ | ✅ | ❌ |
| **Incremental** | ❌ | ❌ | ✅ | ❌ |
| **Compression** | External | ✅ (-z) | ✅ (lz4/zstd) | ✅ (gzip) |
| **Encryption** | External | ❌ | ✅ Built-in | External |
| **Parallel** | ✅ (-j) | ✅ | ✅ | Limited |
| **S3/Cloud** | External | ❌ | ✅ Native | Limited |
| **Multi-server** | Manual | Manual | Multiple stanzas | ✅ Designed for |
| **Retention mgmt** | Manual | Manual | ✅ Auto | ✅ Auto |
| **Backup verify** | Manual | ✅ | ✅ | ✅ |
| **Cross-version** | ✅ | ❌ | ❌ | ❌ |
| **Selective restore** | ✅ (table) | ❌ | ❌ | ❌ |
| **Complexity** | Low | Low | Medium | High |
| **Best for** | Migration, dev | Standby setup | Production | Enterprise multi-server |

**Decision Guide:**
```
Development/Testing:
  → pg_dump (simple, portable)

Standby server setup:
  → pg_basebackup (one-time, Replica setup এ)

Production single server:
  → pgBackRest (Full/Diff/Incr + PITR + encryption + S3)

Production multiple servers:
  → pgBackRest (multiple stanzas) বা Barman (centralized)

Database migration (version upgrade):
  → pg_dump + pg_restore (cross-version)

Emergency single table restore:
  → pg_dump --table=tablename

CI/CD test database:
  → pg_dump (script করা সহজ)
```

---


**Hands-on: তোমার Environment এ কোন Tool কাজ করছে Test করো**

```bash
# ─── pg_dump test ───
echo "=== Testing pg_dump ==="
time pg_dump -U postgres -Fc mydb > /tmp/test_dump.dump
echo "Dump size: $(du -sh /tmp/test_dump.dump)"
pg_restore --list /tmp/test_dump.dump | wc -l
echo "Objects in dump: $(pg_restore --list /tmp/test_dump.dump | wc -l)"

# ─── pg_basebackup test ───
echo ""
echo "=== Testing pg_basebackup ==="
time pg_basebackup -U replicator -h localhost \
    -D /tmp/test_basebackup \
    --checkpoint=fast -P -Xs
echo "Basebackup size: $(du -sh /tmp/test_basebackup)"
ls /tmp/test_basebackup/
rm -rf /tmp/test_basebackup

# ─── pgBackRest test ───
echo ""
echo "=== Testing pgBackRest ==="
sudo -u postgres pgbackrest --stanza=main check
sudo -u postgres pgbackrest --stanza=main --type=full backup
sudo -u postgres pgbackrest --stanza=main info

# ─── All tools version check ───
echo ""
echo "=== Tool Versions ==="
pg_dump --version
pg_basebackup --version
pgbackrest version 2>/dev/null || echo "pgBackRest not installed"
barman --version 2>/dev/null || echo "Barman not installed"

# ─── Backup সব tools এ একসাথে run time compare করো ───
echo ""
echo "=== Backup Time Comparison ==="
echo "pg_dump:"
time pg_dump -U postgres -Fc --compress=9 mydb > /dev/null

echo "pgBackRest incremental:"
time sudo -u postgres pgbackrest --stanza=main --type=incr backup
```

```bash
# ─── Production এ কোনটা choose করবে সেটা নিজেই decide করো ───
DB_SIZE=$(psql -U postgres -t -A -c "SELECT pg_size_pretty(pg_database_size('mydb'));")
echo "Database size: $DB_SIZE"

# Tools installed কোনগুলো?
echo "Available tools:"
which pg_dump      && echo "  ✅ pg_dump"
which pgbackrest   && echo "  ✅ pgBackRest" || echo "  ❌ pgBackRest (install: dnf install pgbackrest)"
which barman       && echo "  ✅ Barman" || echo "  ❌ Barman (install: dnf install barman)"

# WAL archiving configured?
ARCHIVE=$(psql -U postgres -t -A -c "SHOW archive_mode;")
echo "WAL archiving: $ARCHIVE"
[ "$ARCHIVE" = "on" ] && echo "  ✅ PITR সম্ভব" || echo "  ❌ PITR সম্ভব না — archive_mode=on করো"
```

# 8. Monitoring

## 8.1 OS Level

```bash
# CPU — postgres process কতটা নিচ্ছে
top -u postgres
ps aux --sort=-%cpu | grep postgres | head -10

# Memory
free -h
# PostgreSQL process memory:
ps aux --sort=-%mem | grep "postgres:" | head -10

# Disk usage
df -h /var/lib/pgsql            # Data partition
iostat -x 1 5                   # Disk I/O (সব disk)
iotop -o                        # কোন process সবচেয়ে বেশি I/O করছে

# Network (replication এর জন্য)
ss -tnp | grep 5432
netstat -tn | grep 5432
```

### pg_prewarm — Restart পরে Buffer Pool Warm করো

```
Problem: PostgreSQL restart হলে Shared Buffer cache empty হয়।
  First few minutes: সব query disk থেকে read করে → slow
  Production restart: "cold start" effect → spike in disk I/O, slow queries

Solution: pg_prewarm — restart এর পরে important tables/indexes
  Shared Buffer এ আবার load করো
```

```sql
CREATE EXTENSION pg_prewarm;

-- ─── Manual prewarm ───
-- Specific table warm করো
SELECT pg_prewarm('orders');               -- full table
SELECT pg_prewarm('orders', 'buffer');     -- shared buffers এ
SELECT pg_prewarm('orders', 'prefetch');   -- OS page cache এ (faster)
SELECT pg_prewarm('idx_orders_user_id');   -- index warm করো

-- ─── Autoprewarm — restart এ automatically warm করো ───
-- postgresql.conf এ:
-- shared_preload_libraries = 'pg_prewarm'
-- pg_prewarm.autoprewarm = on           (default on if loaded)
-- pg_prewarm.autoprewarm_interval = 300 (5 মিনিটে একবার state save)

-- কীভাবে কাজ করে:
-- ① Running এ: প্রতি 5 min এ current buffer state save করে ($PGDATA/autoprewarm.blocks)
-- ② Restart এ: saved state থেকে automatically load করে background এ
-- ③ Result: restart পরে warm-up time অনেক কমে

-- Manual state save করো (planned restart এর আগে)
SELECT autoprewarm_start_worker();
-- অথবা:
SELECT pg_prewarm(oid::regclass) FROM pg_class
WHERE relkind IN ('r', 'i')
  AND relpages > 1000  -- 1000+ page এর objects
ORDER BY relpages DESC
LIMIT 50;

-- Prewarm status দেখো
SELECT * FROM pg_prewarm_autoprewarm_status();
```

```bash
# postgresql.conf এ enable করো
psql -U postgres -c "
ALTER SYSTEM SET shared_preload_libraries = 'pg_stat_statements,pg_prewarm';
"
# Restart দরকার (shared_preload_libraries)
sudo systemctl restart postgresql-16

# Verify autoprewarm চলছে কিনা
psql -U postgres -c "
SELECT name, setting FROM pg_settings
WHERE name LIKE '%prewarm%';"
# pg_prewarm.autoprewarm → on
```

---

## 8.2 pg_stat_activity — Connection এবং Query Monitoring

```sql
-- সব active connections
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    state,
    wait_event_type,
    wait_event,
    now() - xact_start  AS transaction_age,
    now() - query_start AS query_age,
    LEFT(query, 100)    AS current_query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY transaction_age DESC NULLS LAST;

-- Connection count by state
SELECT state, count(*)
FROM pg_stat_activity
GROUP BY state
ORDER BY count DESC;

-- Connection utilization (80%+ হলে pgBouncer দরকার)
SELECT
    count(*) AS active_connections,
    (SELECT setting::int FROM pg_settings WHERE name = 'max_connections') AS max_allowed,
    ROUND(count(*) * 100.0 /
        (SELECT setting::int FROM pg_settings WHERE name = 'max_connections'), 2) AS utilization_pct
FROM pg_stat_activity;

-- Idle in transaction — বিপজ্জনক! Lock ধরে রাখে
SELECT pid, usename, now() - xact_start AS idle_duration, LEFT(query, 80)
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY idle_duration DESC;
-- এদের kill করো দরকার হলে:
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND now() - xact_start > INTERVAL '10 minutes';

-- Long running queries
SELECT pid, now() - query_start AS duration, state, LEFT(query, 80)
FROM pg_stat_activity
WHERE now() - query_start > INTERVAL '5 minutes'
  AND state != 'idle'
ORDER BY duration DESC;

-- Wait events — কী কারণে wait করছে
SELECT wait_event_type, wait_event, count(*)
FROM pg_stat_activity
WHERE wait_event IS NOT NULL
GROUP BY wait_event_type, wait_event
ORDER BY count DESC;
```

---

## 8.3 pg_stat_user_tables — Table Health

```sql
-- Table access statistics + VACUUM status
SELECT
    relname                                                            AS table_name,
    n_live_tup                                                         AS live_rows,
    n_dead_tup                                                         AS dead_rows,
    ROUND(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct,
    seq_scan,
    idx_scan,
    ROUND(idx_scan * 100.0 / NULLIF(seq_scan + idx_scan, 0), 2)       AS idx_scan_pct,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables
ORDER BY dead_pct DESC NULLS LAST;

-- Tables needing VACUUM urgently (>10% dead)
SELECT relname, n_dead_tup, n_live_tup,
    ROUND(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct,
    last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
  AND n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0) > 10
ORDER BY dead_pct DESC;

-- Table এবং index sizes
SELECT
    t.tablename,
    pg_size_pretty(pg_relation_size(c.oid))                              AS table_size,
    pg_size_pretty(pg_indexes_size(c.oid))                               AS indexes_size,
    pg_size_pretty(pg_total_relation_size(c.oid))                        AS total_size
FROM pg_tables t
JOIN pg_class c ON c.relname = t.tablename AND c.relnamespace = 'public'::regnamespace
WHERE t.schemaname = 'public'
ORDER BY pg_total_relation_size(c.oid) DESC
LIMIT 20;

-- Table bloat estimate
SELECT
    schemaname, tablename,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size,
    n_dead_tup,
    ROUND(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS bloat_pct
FROM pg_stat_user_tables
WHERE pg_relation_size(schemaname||'.'||tablename) > 10 * 1024 * 1024  -- 10MB+
ORDER BY bloat_pct DESC NULLS LAST;
```

---

## 8.4 Replication Monitoring

```sql
-- Primary এ: সব Replica এর status
SELECT
    application_name,
    client_addr,
    state,                  -- streaming/catchup/backup
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    pg_size_pretty(sent_lsn - replay_lsn) AS total_lag_bytes,
    write_lag,              -- Network lag
    flush_lag,              -- Disk write lag (Replica)
    replay_lag,             -- Apply lag (Replica)
    sync_state              -- async/sync/quorum
FROM pg_stat_replication;

-- Replica এ: Recovery/Replication status
SELECT
    pg_is_in_recovery()                                AS is_standby,
    pg_last_wal_receive_lsn()                          AS last_received_lsn,
    pg_last_wal_replay_lsn()                           AS last_replayed_lsn,
    pg_last_xact_replay_timestamp()                    AS last_replay_time,
    now() - pg_last_xact_replay_timestamp()            AS replication_delay,
    pg_is_wal_replay_paused()                          AS replay_paused;

-- Replication slots (unused slot WAL জমিয়ে disk full করে!)
SELECT
    slot_name,
    slot_type,
    active,
    restart_lsn,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS wal_retained
FROM pg_replication_slots;
-- WARNING: active=false এবং wal_retained বড় হলে slot drop করো
-- SELECT pg_drop_replication_slot('slot_name');
```

---

## 8.5 VACUUM Monitoring

```sql
-- Currently running VACUUM এর progress
SELECT
    p.pid,
    relid::regclass                                                AS table_name,
    phase,                  -- scanning heap, vacuuming indexes, etc.
    heap_blks_total,
    heap_blks_scanned,
    ROUND(heap_blks_scanned * 100.0 / NULLIF(heap_blks_total, 0), 2) AS progress_pct,
    index_vacuum_count,
    num_dead_tuples
FROM pg_stat_progress_vacuum p
JOIN pg_stat_activity a ON p.pid = a.pid;

-- XID wraparound risk (সবচেয়ে critical metric!)
SELECT
    datname,
    age(datfrozenxid)           AS xid_age,
    2000000000 - age(datfrozenxid) AS safe_transactions_left
FROM pg_database
ORDER BY xid_age DESC;
-- safe_transactions_left < 200 million → জরুরি action নাও!
-- VACUUM FREEZE <table>; চালাও

-- Table level XID age
SELECT schemaname, relname,
    age(relfrozenxid)                           AS table_xid_age,
    n_dead_tup, n_live_tup,
    last_autovacuum,
    now() - last_autovacuum                     AS since_last_vacuum
FROM pg_stat_user_tables
JOIN pg_class c ON c.relname = relname
ORDER BY age(relfrozenxid) DESC
LIMIT 20;
```

---

## 8.6 Lock Monitoring

```sql
-- Blocked queries + who is blocking them
SELECT
    blocked.pid                         AS blocked_pid,
    blocked.usename                     AS blocked_user,
    now() - blocked.xact_start         AS blocked_duration,
    LEFT(blocked.query, 80)             AS blocked_query,
    blocking.pid                        AS blocking_pid,
    blocking.usename                    AS blocking_user,
    LEFT(blocking.query, 80)            AS blocking_query
FROM pg_stat_activity AS blocked
JOIN pg_stat_activity AS blocking
    ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE cardinality(pg_blocking_pids(blocked.pid)) > 0
ORDER BY blocked_duration DESC;

-- All current locks
SELECT
    pid,
    locktype,
    relation::regclass AS table_name,
    mode,
    granted,
    LEFT(query, 60)    AS query
FROM pg_locks l
JOIN pg_stat_activity a USING (pid)
WHERE relation IS NOT NULL
ORDER BY granted, pid;
```

---

## 8.7 Index Usage Monitoring

```sql
-- Unused indexes (never used since server start)
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan                                              AS times_used,
    pg_size_pretty(pg_relation_size(indexrelid))          AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexname NOT LIKE '%pkey%'
  AND indexname NOT LIKE '%unique%'
ORDER BY pg_relation_size(indexrelid) DESC;
-- → Drop করো: DROP INDEX CONCURRENTLY idx_name;

-- Seq scan বেশি হচ্ছে এমন table (index দরকার?)
SELECT relname, seq_scan, idx_scan,
    ROUND(idx_scan * 100.0 / NULLIF(seq_scan + idx_scan, 0), 2) AS idx_pct
FROM pg_stat_user_tables
WHERE seq_scan > 100  -- significant scan count
ORDER BY seq_scan DESC
LIMIT 10;

-- Index bloat check
SELECT
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC
LIMIT 20;
-- Bloated index → REINDEX CONCURRENTLY idx_name;
```

---
---

## 8.7b amcheck — Index Corruption Detection

Production এ index corrupt হলে query wrong result দিতে পারে বা crash হতে পারে। `amcheck` দিয়ে B-Tree index এর integrity check করো।

```sql
CREATE EXTENSION amcheck;

-- ─── Single index check ───
SELECT bt_index_check('idx_users_email');
-- Error দিলে → index corrupt
-- No error দিলে → OK

-- ─── Thorough check (heap ও check করে) ───
SELECT bt_index_parent_check('idx_users_email', true);
-- true = heap cross-check করো
-- Slower কিন্তু more complete

-- ─── সব indexes check করো ───
DO $$
DECLARE
    idx RECORD;
BEGIN
    FOR idx IN
        SELECT indexrelid::regclass AS index_name
        FROM pg_index i
        JOIN pg_class c ON c.oid = i.indrelid
        JOIN pg_am a ON a.oid = (SELECT relam FROM pg_class WHERE oid = i.indexrelid)
        WHERE a.amname = 'btree'
          AND c.relnamespace = 'public'::regnamespace
    LOOP
        BEGIN
            PERFORM bt_index_check(idx.index_name);
            RAISE NOTICE '✅ OK: %', idx.index_name;
        EXCEPTION WHEN OTHERS THEN
            RAISE WARNING '❌ CORRUPT: % — %', idx.index_name, SQLERRM;
        END;
    END LOOP;
END $$;

-- ─── Hands-on: Weekly corruption check script ───
```

```bash
cat > /usr/local/bin/index_integrity_check.sh << 'SCRIPT'
#!/bin/bash
DB=${1:-mydb}
echo "=== Index Integrity Check: $DB — $(date) ==="

psql -U postgres -d $DB -t << 'SQL'
CREATE EXTENSION IF NOT EXISTS amcheck;

DO $$
DECLARE
    idx RECORD;
    corrupt_count INT := 0;
BEGIN
    FOR idx IN
        SELECT i.indexrelid::regclass AS index_name
        FROM pg_index i
        JOIN pg_class c ON c.oid = i.indrelid
        JOIN pg_am a ON a.oid = (SELECT relam FROM pg_class WHERE oid = i.indexrelid)
        WHERE a.amname = 'btree'
          AND c.relnamespace = 'public'::regnamespace
          AND NOT i.indisvalid = false  -- INVALID indexes skip করো
    LOOP
        BEGIN
            PERFORM bt_index_check(idx.index_name);
        EXCEPTION WHEN OTHERS THEN
            RAISE WARNING 'CORRUPT INDEX: %', idx.index_name;
            corrupt_count := corrupt_count + 1;
        END;
    END LOOP;
    
    IF corrupt_count = 0 THEN
        RAISE NOTICE 'All indexes OK';
    ELSE
        RAISE WARNING '% corrupt index(es) found!', corrupt_count;
    END IF;
END $$;
SQL
SCRIPT
chmod +x /usr/local/bin/index_integrity_check.sh

# Weekly cron
echo "0 3 * * 0 postgres /usr/local/bin/index_integrity_check.sh mydb >> /var/log/index_check.log 2>&1" | \
    sudo tee /etc/cron.d/index_integrity_check
```

---

## 8.7c Operation Progress Monitoring

বড় table এ `CREATE INDEX`, `VACUUM`, `ANALYZE`, `CLUSTER` কতক্ষণ লাগবে বোঝা যায় `pg_stat_progress_*` views থেকে।

```sql
-- ─── CREATE INDEX CONCURRENTLY progress ───
SELECT
    p.pid,
    p.phase,              -- initializing, waiting for writers, building index, ...
    p.blocks_done,
    p.blocks_total,
    ROUND(p.blocks_done * 100.0 / NULLIF(p.blocks_total, 0), 1) AS pct_done,
    p.tuples_done,
    p.tuples_total,
    now() - a.query_start AS elapsed,
    -- Estimated time remaining
    CASE WHEN p.blocks_done > 0
        THEN ((now() - a.query_start) * (p.blocks_total - p.blocks_done) / p.blocks_done)
    END AS estimated_remaining,
    LEFT(a.query, 80) AS query
FROM pg_stat_progress_create_index p
JOIN pg_stat_activity a ON p.pid = a.pid;

-- ─── VACUUM progress ───
SELECT
    p.pid,
    relid::regclass AS table_name,
    p.phase,
    p.heap_blks_total,
    p.heap_blks_scanned,
    ROUND(p.heap_blks_scanned * 100.0 / NULLIF(p.heap_blks_total, 0), 1) AS scan_pct,
    p.heap_blks_vacuumed,
    ROUND(p.heap_blks_vacuumed * 100.0 / NULLIF(p.heap_blks_total, 0), 1) AS vacuum_pct,
    p.index_vacuum_count,
    p.num_dead_tuples,
    now() - a.query_start AS elapsed
FROM pg_stat_progress_vacuum p
JOIN pg_stat_activity a ON p.pid = a.pid;

-- ─── ANALYZE progress ───
SELECT
    p.pid,
    relid::regclass AS table_name,
    p.phase,
    p.sample_blks_total,
    p.sample_blks_scanned,
    ROUND(p.sample_blks_scanned * 100.0 / NULLIF(p.sample_blks_total, 0), 1) AS pct_done,
    now() - a.query_start AS elapsed
FROM pg_stat_progress_analyze p
JOIN pg_stat_activity a ON p.pid = a.pid;

-- ─── CLUSTER / VACUUM FULL progress ───
SELECT
    p.pid,
    relid::regclass AS table_name,
    p.phase,
    p.cluster_index_relid::regclass AS cluster_index,
    p.heap_blks_total,
    p.heap_blks_scanned,
    ROUND(p.heap_blks_scanned * 100.0 / NULLIF(p.heap_blks_total, 0), 1) AS pct_done,
    now() - a.query_start AS elapsed
FROM pg_stat_progress_cluster p
JOIN pg_stat_activity a ON p.pid = a.pid;

-- ─── pg_repack progress (এ নিজস্ব view নেই, query দিয়ে দেখো) ───
SELECT pid, now()-query_start AS elapsed, LEFT(query, 80)
FROM pg_stat_activity
WHERE query LIKE '%repack%' AND state = 'active';

-- ─── সব running maintenance operations একসাথে ───
SELECT 'CREATE INDEX' AS operation, pid, phase,
    ROUND(blocks_done*100.0/NULLIF(blocks_total,0),1)||'%' AS progress
FROM pg_stat_progress_create_index
UNION ALL
SELECT 'VACUUM', pid, phase,
    ROUND(heap_blks_vacuumed*100.0/NULLIF(heap_blks_total,0),1)||'%'
FROM pg_stat_progress_vacuum
UNION ALL
SELECT 'ANALYZE', pid, phase,
    ROUND(sample_blks_scanned*100.0/NULLIF(sample_blks_total,0),1)||'%'
FROM pg_stat_progress_analyze
UNION ALL
SELECT 'CLUSTER/VACUUM FULL', pid, phase,
    ROUND(heap_blks_scanned*100.0/NULLIF(heap_blks_total,0),1)||'%'
FROM pg_stat_progress_cluster;
```

```bash
# ─── Real-time progress watch ───
watch -n 3 'psql -U postgres -t -c "
SELECT operation, pid, phase, progress FROM (
    SELECT '"'"'CREATE INDEX'"'"' AS operation, pid, phase,
        ROUND(blocks_done*100.0/NULLIF(blocks_total,0),1)||'"'"'%'"'"' AS progress
    FROM pg_stat_progress_create_index
    UNION ALL
    SELECT '"'"'VACUUM'"'"', pid, phase,
        ROUND(heap_blks_vacuumed*100.0/NULLIF(heap_blks_total,0),1)||'"'"'%'"'"'
    FROM pg_stat_progress_vacuum
    UNION ALL
    SELECT '"'"'ANALYZE'"'"', pid, phase,
        ROUND(sample_blks_scanned*100.0/NULLIF(sample_blks_total,0),1)||'"'"'%'"'"'
    FROM pg_stat_progress_analyze
) t ORDER BY operation;"'
# Ctrl+C দিয়ে বের হও
```

---


## 8.8 Percona Monitoring and Management (PMM)

PMM হলো Percona এর open-source monitoring platform। MySQL এবং PostgreSQL দুটোই monitor করে। Grafana dashboard + Query Analytics (QAN) built-in।

### Architecture

```
  ┌───────────────────────────────────────────────────┐
  │                  PMM Server                       │
  │  Grafana UI · Prometheus · VictoriaMetrics        │
  │  Query Analytics (QAN) · Alertmanager             │
  │  HTTPS Port 443                                   │
  └──────────────────────┬────────────────────────────┘
                         │ metrics
       ┌─────────────────┼─────────────────┐
       ▼                 ▼                 ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │PMM Client│    │PMM Client│    │PMM Client│
  │PG Node 1 │    │PG Node 2 │    │MySQL     │
  │pg_exporter│   │pg_exporter│   │(works too)│
  │node_export│   │node_export│   │          │
  └──────────┘    └──────────┘    └──────────┘
```

### PMM Server Install (Docker)

```bash
# Docker install
sudo dnf install -y docker-ce docker-ce-cli
sudo systemctl start docker && sudo systemctl enable docker

# PMM Server
docker run -d \
    --name pmm-server \
    --restart always \
    -p 80:80 \
    -p 443:443 \
    -v pmm-data:/srv \
    percona/pmm-server:2

# ~60 সেকেন্ড পর browser এ:
# https://your-server-ip
# Login: admin / admin → password change করো
```

### PMM Client Install (PostgreSQL Server এ)

```bash
# Install
sudo dnf install -y https://repo.percona.com/yum/percona-release-latest.noarch.rpm
sudo percona-release enable pmm2-client
sudo dnf install -y pmm2-client

# PMM Server এ register করো
sudo pmm-admin config \
    --server-insecure-tls \
    --server-url=https://admin:YourPassword@172.16.93.150

# PostgreSQL service add করো
sudo pmm-admin add postgresql \
    --username=pmm_monitor \
    --password=PMM@Monitor2024! \
    --service-name=pg-primary \
    --host=127.0.0.1 \
    --port=5432 \
    --query-source=pgstatmonitor    # pg_stat_monitor ব্যবহার করো

# Status দেখো
sudo pmm-admin list
```

```sql
-- PostgreSQL এ monitoring user তৈরি করো
CREATE USER pmm_monitor PASSWORD 'PMM@Monitor2024!';
GRANT pg_monitor TO pmm_monitor;
GRANT SELECT ON pg_stat_monitor TO pmm_monitor;   -- pg_stat_monitor আছে হলে

-- pg_hba.conf এ:
-- host all pmm_monitor 127.0.0.1/32 scram-sha-256
```

### PMM Dashboard Overview

```
PostgreSQL Overview:
  → Active connections, TPS, Queries/sec
  → Shared buffers hit rate
  → Temp file usage, WAL generation rate

PostgreSQL Instance Summary:
  → Database size growth, Table bloat
  → Dead tuples per table, Autovacuum activity

PostgreSQL Replication:
  → Streaming status, Lag in bytes এবং time
  → Replication slot WAL retention

Node Summary (OS):
  → CPU, Memory, Disk I/O, Network

Query Analytics (QAN):
  → Top queries by load/time/calls
  → Per-query execution plan (from pg_stat_monitor)
  → Response time histogram (P50/P95/P99)
  → Per-user, per-client-IP breakdown
```

### PMM Alerting

```
Built-in Alert Templates:
  - Replication lag > threshold
  - Connection utilization > 80%
  - Shared buffer hit rate < 95%
  - XID wraparound risk (age > 1.5B)
  - Autovacuum not running
  - Disk usage > 85%
  - Long running queries > N minutes
  - Bloat > threshold

Alert channels:
  Email, Slack, PagerDuty, OpsGenie
```

---

## 8.9 Monitoring Dashboard Script

```bash
#!/bin/bash
# /usr/local/bin/pg_health_check.sh
PSQL="psql -U postgres -t -A -c"

echo "========================================="
echo " PostgreSQL Health Check - $(date)"
echo "========================================="

echo ""
echo "--- VERSION ---"
$PSQL "SELECT version();"

echo ""
echo "--- CONNECTIONS ---"
$PSQL "SELECT state, count(*) FROM pg_stat_activity GROUP BY state ORDER BY count DESC;"
$PSQL "SELECT count(*) AS total_connections,
    (SELECT setting FROM pg_settings WHERE name='max_connections') AS max_allowed
    FROM pg_stat_activity;"

echo ""
echo "--- BUFFER HIT RATE (99%+ হওয়া উচিত) ---"
$PSQL "SELECT ROUND(sum(heap_blks_hit) * 100.0 /
    NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0), 2) AS hit_rate_pct
    FROM pg_statio_user_tables;"

echo ""
echo "--- LONG RUNNING QUERIES (>5min) ---"
$PSQL "SELECT pid, now()-query_start AS duration, state, LEFT(query, 80)
    FROM pg_stat_activity
    WHERE now()-query_start > INTERVAL '5 min' AND state != 'idle'
    ORDER BY duration DESC
    LIMIT 5;"

echo ""
echo "--- IDLE IN TRANSACTION ---"
$PSQL "SELECT count(*) AS idle_in_transaction
    FROM pg_stat_activity WHERE state = 'idle in transaction';"

echo ""
echo "--- REPLICATION STATUS ---"
$PSQL "SELECT application_name, state,
    pg_size_pretty(sent_lsn - replay_lsn) AS lag_bytes,
    replay_lag FROM pg_stat_replication;" 2>/dev/null \
|| $PSQL "SELECT pg_is_in_recovery() AS is_standby,
    now() - pg_last_xact_replay_timestamp() AS lag;"

echo ""
echo "--- TOP DEAD TUPLE TABLES ---"
$PSQL "SELECT relname,
    n_dead_tup, n_live_tup,
    ROUND(n_dead_tup*100.0/NULLIF(n_live_tup+n_dead_tup,0),2) AS dead_pct
    FROM pg_stat_user_tables
    WHERE n_dead_tup > 1000
    ORDER BY dead_pct DESC NULLS LAST LIMIT 5;"

echo ""
echo "--- XID WRAPAROUND RISK ---"
$PSQL "SELECT datname, age(datfrozenxid) AS xid_age,
    2000000000 - age(datfrozenxid) AS safe_left
    FROM pg_database ORDER BY xid_age DESC LIMIT 3;"

echo ""
echo "--- DISK USAGE ---"
df -h /var/lib/pgsql | tail -1
$PSQL "SELECT pg_size_pretty(pg_database_size(current_database())) AS current_db_size;"

echo ""
echo "--- ARCHIVE STATUS ---"
$PSQL "SELECT last_archived_wal, last_archived_time,
    failed_count, last_failed_time
    FROM pg_stat_archiver;"

echo "========================================="
```

```bash
chmod +x /usr/local/bin/pg_health_check.sh

# Cron এ প্রতি 5 মিনিটে
echo "*/5 * * * * postgres /usr/local/bin/pg_health_check.sh >> /var/log/pg_health.log 2>&1" \
    | sudo tee /etc/cron.d/pg_health_check
```

---

## 8.10 Key Metrics Thresholds

| Metric | Healthy | Warning | Critical | Action |
|---|---|---|---|---|
| Buffer Hit Rate | > 99% | 95-99% | < 95% | shared_buffers বাড়াও |
| Connection Utilization | < 70% | 70-85% | > 85% | pgBouncer যোগ করো |
| Replication Lag (bytes) | < 1MB | 1-100MB | > 100MB | Network/Load check |
| Replication Lag (time) | < 5s | 5-60s | > 60s | Replica load check |
| Dead Tuple % | < 5% | 5-20% | > 20% | VACUUM/autovacuum tune |
| XID Age | < 500M | 500M-1.5B | > 1.5B | VACUUM FREEZE জরুরি! |
| Disk Usage | < 70% | 70-85% | > 85% | Space বাড়াও |
| Idle in Transaction | 0 | 1-5 | > 5 | idle_in_transaction_timeout |
| Archive Failures | 0 | 1-5 | > 5 | Archive command fix করো |
| Temp File Usage | 0 | < 100MB | > 1GB | work_mem বাড়াও |
| Checkpoint Warning | None | Occasional | Frequent | max_wal_size বাড়াও |
| Bloat % per table | < 10% | 10-30% | > 30% | pg_repack / autovacuum |
| Long Queries (>5min) | 0 | 1-3 | > 3 | Investigate + kill |

```ini
# Idle in transaction timeout (postgresql.conf)
idle_in_transaction_session_timeout = '10min'
# এই সময়ের বেশি idle in transaction → auto-terminate

statement_timeout = '30min'
# Single statement এর maximum execution time
# OLTP: '30s', Analytics: '30min' বা off
```

---

**Hands-on: সব Key Metrics একসাথে Check করো**

```sql
-- ─── Complete Health Check Query ───
SELECT '=== CONNECTION STATUS ===' AS section, '' AS value, '' AS status
UNION ALL
SELECT 'Active connections',
    count(*)::text,
    CASE WHEN count(*) * 100 / (SELECT setting::int FROM pg_settings WHERE name='max_connections') > 85
         THEN '⚠ CRITICAL' WHEN count(*) * 100 / (SELECT setting::int FROM pg_settings WHERE name='max_connections') > 70
         THEN '! WARNING' ELSE '✓ OK' END
FROM pg_stat_activity

UNION ALL
SELECT '=== BUFFER HIT RATE ===' , '', ''

UNION ALL
SELECT 'Buffer hit rate',
    ROUND(sum(heap_blks_hit)*100.0/NULLIF(sum(heap_blks_hit)+sum(heap_blks_read),0),2)::text || '%',
    CASE WHEN ROUND(sum(heap_blks_hit)*100.0/NULLIF(sum(heap_blks_hit)+sum(heap_blks_read),0),2) < 95
         THEN '⚠ CRITICAL' WHEN ROUND(sum(heap_blks_hit)*100.0/NULLIF(sum(heap_blks_hit)+sum(heap_blks_read),0),2) < 99
         THEN '! WARNING' ELSE '✓ OK' END
FROM pg_statio_user_tables

UNION ALL
SELECT '=== XID WRAPAROUND ===' , '', ''

UNION ALL
SELECT 'Max XID age',
    MAX(age(datfrozenxid))::text,
    CASE WHEN MAX(age(datfrozenxid)) > 1500000000 THEN '⚠ CRITICAL'
         WHEN MAX(age(datfrozenxid)) > 500000000  THEN '! WARNING'
         ELSE '✓ OK' END
FROM pg_database

UNION ALL
SELECT '=== DEAD TUPLES ===' , '', ''

UNION ALL
SELECT 'Tables with >10% dead tuples',
    COUNT(*)::text,
    CASE WHEN COUNT(*) > 0 THEN '! WARNING: run VACUUM' ELSE '✓ OK' END
FROM pg_stat_user_tables
WHERE n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0) > 10

UNION ALL
SELECT '=== LONG QUERIES ===' , '', ''

UNION ALL
SELECT 'Queries running >5 min',
    COUNT(*)::text,
    CASE WHEN COUNT(*) > 0 THEN '! WARNING: investigate' ELSE '✓ OK' END
FROM pg_stat_activity
WHERE now() - query_start > INTERVAL '5 min' AND state = 'active'

UNION ALL
SELECT '=== ARCHIVE STATUS ===' , '', ''

UNION ALL
SELECT 'Archive failures',
    failed_count::text,
    CASE WHEN failed_count > 0 THEN '⚠ CRITICAL: fix archive' ELSE '✓ OK' END
FROM pg_stat_archiver;
```

```bash
# ─── OS Level Metrics এক নজরে ───
echo "=== Disk Usage ==="
df -h /var/lib/pgsql | awk 'NR>1 {print "Used: "$5, "Free: "$4}'

echo "=== Memory ==="
free -h | grep Mem | awk '{print "Total:",$2,"Used:",$3,"Free:",$4}'

echo "=== PostgreSQL Process Memory ==="
ps aux --sort=-%mem | grep "postgres:" | awk '{sum+=$4} END {print "Total RSS: " sum "%"}'

echo "=== Disk I/O ==="
iostat -x 1 2 | grep -E "Device|sd|nvme" | tail -5

echo "=== Top PostgreSQL Waits ==="
psql -U postgres -t -c "
SELECT wait_event_type, wait_event, count(*)
FROM pg_stat_activity
WHERE wait_event IS NOT NULL
GROUP BY wait_event_type, wait_event
ORDER BY count DESC LIMIT 5;"
```


## 8.11 pg_stat_database — Database-Level Statistics

```sql
-- সব database এর overall statistics
SELECT
    datname,
    numbackends                                              AS connections,
    xact_commit                                             AS commits,
    xact_rollback                                           AS rollbacks,
    ROUND(xact_rollback * 100.0 / NULLIF(xact_commit + xact_rollback, 0), 2) AS rollback_pct,
    blks_read                                               AS disk_reads,
    blks_hit                                                AS buffer_hits,
    ROUND(blks_hit * 100.0 / NULLIF(blks_read + blks_hit, 0), 2) AS hit_rate_pct,
    tup_returned                                            AS rows_returned,
    tup_fetched                                             AS rows_fetched,
    tup_inserted                                            AS rows_inserted,
    tup_updated                                             AS rows_updated,
    tup_deleted                                             AS rows_deleted,
    temp_files                                              AS temp_files_created,
    pg_size_pretty(temp_bytes)                              AS temp_data_written,
    deadlocks,
    conflicts,                   -- Standby query conflicts
    pg_size_pretty(pg_database_size(datname)) AS db_size,
    stats_reset
FROM pg_stat_database
WHERE datname NOT IN ('template0', 'template1')
ORDER BY pg_database_size(datname) DESC;

-- Database এর write pattern
SELECT datname,
    tup_inserted + tup_updated + tup_deleted AS total_writes,
    tup_inserted AS inserts,
    tup_updated  AS updates,
    tup_deleted  AS deletes
FROM pg_stat_database
WHERE datname = current_database();

-- Temp file usage (work_mem কম হলে বাড়ে)
SELECT datname, temp_files, pg_size_pretty(temp_bytes) AS temp_size
FROM pg_stat_database
WHERE temp_files > 0
ORDER BY temp_bytes DESC;
-- Temp files বেশি হলে → work_mem বাড়াও বা slow query optimize করো

-- Deadlock tracking
SELECT datname, deadlocks
FROM pg_stat_database
WHERE deadlocks > 0;
-- Deadlock > 0 → Application এ lock order inconsistency আছে

-- Statistics reset করো (fresh monitoring শুরু করতে)
SELECT pg_stat_reset();  -- Current database
```

---

## 8.12 pg_stat_bgwriter — Background Writer Monitoring

```sql
-- Background writer এবং checkpoint statistics
SELECT
    checkpoints_timed,           -- Timeout এ triggered checkpoint
    checkpoints_req,             -- Manually বা wal_size এ triggered
    checkpoint_write_time,       -- ms spent writing dirty pages
    checkpoint_sync_time,        -- ms spent syncing to disk
    buffers_checkpoint,          -- Checkpoint এ written buffers
    buffers_clean,               -- bgwriter এ written buffers
    maxwritten_clean,            -- bgwriter stopped (rate limit hit) count
    buffers_backend,             -- Backend processes directly wrote (bad!)
    buffers_backend_fsync,       -- Backend এ fsync করতে হয়েছে (very bad!)
    buffers_alloc,               -- New buffers allocated
    stats_reset
FROM pg_stat_bgwriter;

-- কীভাবে interpret করবো:
-- checkpoints_req >> checkpoints_timed → max_wal_size বাড়াও
-- buffers_backend বেশি → bgwriter enough fast না, bgwriter_lru_multiplier বাড়াও
-- maxwritten_clean বেশি → bgwriter_lru_maxpages বাড়াও
-- checkpoint_sync_time বেশি → SSD দরকার বা fsync tuning

-- Checkpoint efficiency
SELECT
    checkpoints_req + checkpoints_timed AS total_checkpoints,
    ROUND(checkpoints_req * 100.0 / NULLIF(checkpoints_req + checkpoints_timed, 0), 2) AS forced_pct,
    ROUND(buffers_checkpoint * 100.0 / NULLIF(buffers_checkpoint + buffers_clean + buffers_backend, 0), 2) AS checkpoint_write_pct
FROM pg_stat_bgwriter;
-- forced_pct > 20% → checkpoints too frequent → max_wal_size বাড়াও
```

---

---

## 8.12b pg_stat_io — PostgreSQL 16 এর I/O Statistics

`pg_stat_io` হলো PostgreSQL 16 এর নতুন view। আগে I/O statistics পেতে OS level এ যেতে হতো বা `pg_statio_user_tables` এ limited data ছিল। এখন buffer read/write, WAL I/O সব একজায়গায়।

```sql
-- ─── Overall I/O summary ───
SELECT
    backend_type,
    object,           -- relation, index, toast, wal, temp relation
    context,          -- normal, vacuum, bulkread, bulkwrite
    reads,            -- heap/index pages read from disk
    read_time,        -- ms spent reading
    writes,           -- pages written
    write_time,       -- ms spent writing
    writebacks,       -- pages forced to disk (fsync)
    extends,          -- new pages appended
    hits,             -- shared buffer hits
    evictions,        -- pages evicted from buffer
    reuses,           -- buffer reuse (temp files)
    fsyncs,           -- fsync calls
    fsync_time        -- ms spent in fsync
FROM pg_stat_io
ORDER BY reads DESC NULLS LAST;

-- ─── Buffer hit rate per context ───
SELECT
    backend_type,
    object,
    context,
    ROUND(hits * 100.0 / NULLIF(hits + reads, 0), 2) AS hit_rate_pct,
    reads,
    hits,
    pg_size_pretty(reads * 8192) AS data_read_from_disk
FROM pg_stat_io
WHERE hits + reads > 0
ORDER BY reads DESC
LIMIT 15;

-- ─── Checkpoint I/O impact দেখো ───
SELECT
    context,
    object,
    writes,
    writebacks,
    write_time,
    fsync_time
FROM pg_stat_io
WHERE context IN ('normal', 'vacuum')
  AND writes > 0
ORDER BY writes DESC;

-- ─── track_io_timing enable করো (more detail) ───
-- postgresql.conf:
-- track_io_timing = on    -- read_time, write_time populate হবে
-- (slight overhead, modern hardware এ negligible)
```

```bash
# track_io_timing enable করো (reload যথেষ্ট)
psql -U postgres -c "ALTER SYSTEM SET track_io_timing = on;"
psql -U postgres -c "SELECT pg_reload_conf();"
psql -U postgres -c "SHOW track_io_timing;"

# Reset করো (fresh measurement)
psql -U postgres -c "SELECT pg_stat_reset_shared('io');"

# কিছুক্ষণ পরে দেখো
psql -U postgres -c "
SELECT backend_type, object, reads, hits,
    ROUND(hits*100.0/NULLIF(hits+reads,0),1) AS hit_pct,
    ROUND(read_time::numeric,2) AS read_ms,
    ROUND(write_time::numeric,2) AS write_ms
FROM pg_stat_io
WHERE hits + reads > 0
ORDER BY read_time DESC NULLS LAST
LIMIT 10;"
```

---

## 8.13 Log Analysis

PostgreSQL log থেকে অনেক valuable information পাওয়া যায়।

```bash
# Slow query log এ কোন queries সবচেয়ে বেশিবার আছে
sudo grep "duration:" /var/lib/pgsql/16/data/log/postgresql-$(date +%Y-%m-%d).log | \
    awk '{print $NF " ms: " substr($0, index($0,"statement:"))}' | \
    sort -rn | head -20

# Slowest queries (duration দিয়ে sort)
sudo grep "duration:" /var/lib/pgsql/16/data/log/postgresql-*.log | \
    grep -oP 'duration: \K[0-9.]+' | \
    sort -rn | head -10

# Error count by type
sudo grep -E "ERROR|FATAL|PANIC" /var/lib/pgsql/16/data/log/postgresql-*.log | \
    grep -oP '(ERROR|FATAL|PANIC):.*' | \
    sort | uniq -c | sort -rn | head -20

# Connection errors
sudo grep "connection" /var/lib/pgsql/16/data/log/postgresql-*.log | \
    grep -i "fail\|refused\|timeout" | tail -20

# Checkpoint warnings (too frequent → max_wal_size বাড়াও)
sudo grep "checkpoint" /var/lib/pgsql/16/data/log/postgresql-*.log | \
    grep -i "warning\|frequent" | tail -10

# Autovacuum runs (কোন table vacuum হচ্ছে)
sudo grep "automatic vacuum\|automatic analyze" /var/lib/pgsql/16/data/log/postgresql-*.log | \
    tail -20

# Lock waits
sudo grep "lock\|deadlock" /var/lib/pgsql/16/data/log/postgresql-*.log | \
    grep -v "^--" | tail -20
```

```bash
# pgbadger — log analysis tool (advanced)
sudo dnf install -y pgbadger
# অথবা: sudo pip3 install pgbadger --break-system-packages

# HTML report তৈরি করো
pgbadger /var/lib/pgsql/16/data/log/postgresql-*.log \
    -o /tmp/pg_report.html \
    --format stderr \
    --prefix '%m [%p] %q%u@%d '

# Browser এ open করো — দেখাবে:
# - Top slow queries
# - Most frequent queries  
# - Peak connection times
# - Error summary
# - Checkpoint frequency chart
```

---

## 8.14 Alerting Setup — Custom Alerts

PMM না থাকলে custom alerting script।

```bash
cat > /usr/local/bin/pg_alert.sh << 'SCRIPT'
#!/bin/bash
# PostgreSQL Alerting Script
# Cron: */5 * * * * postgres /usr/local/bin/pg_alert.sh

PSQL="psql -U postgres -t -A -c"
ALERT_LOG="/var/log/pg_alerts.log"
SLACK_WEBHOOK="https://hooks.slack.com/services/YOUR/WEBHOOK/URL"

send_alert() {
    local severity=$1
    local message=$2
    echo "$(date) [$severity] $message" >> $ALERT_LOG
    # Slack notification
    curl -s -X POST -H 'Content-type: application/json' \
        --data "{\"text\":\"[$severity] PostgreSQL Alert: $message\"}" \
        $SLACK_WEBHOOK > /dev/null
}

# 1. Connection utilization
CONN_UTIL=$($PSQL "SELECT ROUND(count(*) * 100.0 / (SELECT setting::int FROM pg_settings WHERE name='max_connections'), 0) FROM pg_stat_activity;")
[ "$CONN_UTIL" -gt 85 ] && send_alert "CRITICAL" "Connection utilization: ${CONN_UTIL}%"
[ "$CONN_UTIL" -gt 70 ] && send_alert "WARNING" "Connection utilization: ${CONN_UTIL}%"

# 2. Replication lag
REPLICA_LAG=$($PSQL "SELECT COALESCE(MAX(EXTRACT(EPOCH FROM replay_lag)), 0) FROM pg_stat_replication;" 2>/dev/null)
[ "$(echo "$REPLICA_LAG > 60" | bc -l)" = "1" ] && send_alert "CRITICAL" "Replica lag: ${REPLICA_LAG}s"
[ "$(echo "$REPLICA_LAG > 10" | bc -l)" = "1" ] && send_alert "WARNING" "Replica lag: ${REPLICA_LAG}s"

# 3. Buffer hit rate
HIT_RATE=$($PSQL "SELECT ROUND(sum(heap_blks_hit)*100.0/NULLIF(sum(heap_blks_hit)+sum(heap_blks_read),0),1) FROM pg_statio_user_tables;")
[ "$(echo "$HIT_RATE < 95" | bc -l)" = "1" ] && send_alert "CRITICAL" "Buffer hit rate: ${HIT_RATE}%"

# 4. XID wraparound
XID_AGE=$($PSQL "SELECT MAX(age(datfrozenxid)) FROM pg_database;")
[ "$XID_AGE" -gt 1500000000 ] && send_alert "CRITICAL" "XID age: $XID_AGE — VACUUM FREEZE needed!"
[ "$XID_AGE" -gt 1000000000 ] && send_alert "WARNING" "XID age: $XID_AGE — Check autovacuum"

# 5. Disk usage
DISK_PCT=$(df /var/lib/pgsql | tail -1 | awk '{print $5}' | tr -d '%')
[ "$DISK_PCT" -gt 85 ] && send_alert "CRITICAL" "Disk usage: ${DISK_PCT}%"
[ "$DISK_PCT" -gt 70 ] && send_alert "WARNING" "Disk usage: ${DISK_PCT}%"

# 6. Long running queries
LONG_QUERIES=$($PSQL "SELECT COUNT(*) FROM pg_stat_activity WHERE now()-query_start > INTERVAL '10 minutes' AND state='active';")
[ "$LONG_QUERIES" -gt 0 ] && send_alert "WARNING" "Long running queries: $LONG_QUERIES (>10min)"

# 7. Deadlocks (last hour)
DEADLOCKS=$($PSQL "SELECT SUM(deadlocks) FROM pg_stat_database WHERE datname NOT IN ('template0','template1');")
[ "$DEADLOCKS" -gt 5 ] && send_alert "WARNING" "Deadlocks detected: $DEADLOCKS"

# 8. Archive failures
ARCHIVE_FAIL=$($PSQL "SELECT failed_count FROM pg_stat_archiver;")
[ "$ARCHIVE_FAIL" -gt 0 ] && send_alert "CRITICAL" "WAL archive failures: $ARCHIVE_FAIL"

# 9. Idle in transaction
IDLE_TXN=$($PSQL "SELECT COUNT(*) FROM pg_stat_activity WHERE state='idle in transaction' AND now()-xact_start > INTERVAL '5 minutes';")
[ "$IDLE_TXN" -gt 3 ] && send_alert "WARNING" "Idle in transaction >5min: $IDLE_TXN sessions"
SCRIPT

chmod +x /usr/local/bin/pg_alert.sh
echo "*/5 * * * * postgres /usr/local/bin/pg_alert.sh" | sudo tee /etc/cron.d/pg_alerts
```

---

# 9. Security

## 9.1 Authentication — pg_hba.conf

MySQL এ user এর সাথেই host লেখা হয় (`'user'@'host'`)। PostgreSQL এ authentication rules আলাদা `pg_hba.conf` file এ।

```
# TYPE    DATABASE  USER        ADDRESS           METHOD
# ─────── ─────────────────────────────────────── ──────────────────
local     all       postgres                      peer
# local = Unix socket, peer = OS username = PG username match

local     all       all                           scram-sha-256
# local users এর জন্য password required

host      all       all         127.0.0.1/32      scram-sha-256
# localhost TCP এ scram-sha-256

host      all       all         172.16.93.0/24    scram-sha-256
# Internal network

hostssl   all       all         10.0.0.0/8        scram-sha-256
# SSL required for this range

host      replication replicator 172.16.93.0/24   scram-sha-256
# Replication connections

# যা থাকা উচিত না:
# host all all 0.0.0.0/0 trust   ← কখনো না! Password ছাড়া সব allow
# host all all 0.0.0.0/0 md5     ← md5 weak, scram-sha-256 ব্যবহার করো
```

**TYPE:**
| Type | মানে |
|---|---|
| `local` | Unix socket (same server) |
| `host` | TCP/IP (SSL বা non-SSL) |
| `hostssl` | TCP/IP, SSL required |
| `hostnossl` | TCP/IP, non-SSL only |

**METHOD:**
| Method | Security | কখন |
|---|---|---|
| `trust` | কোনো password নেই | কখনো production এ না! |
| `reject` | সবসময় reject | Block করতে |
| `md5` | MD5 hash | Legacy, avoid |
| `scram-sha-256` | Strong hash | Production default |
| `peer` | OS username match | Local connections |
| `cert` | SSL certificate | High security |

```bash
# Reload (restart ছাড়া pg_hba.conf change apply)
sudo systemctl reload postgresql-16
# অথবা:
psql -c "SELECT pg_reload_conf();"
```

---

## 9.2 Roles & Privileges

PostgreSQL এ **User** এবং **Role** একই জিনিস। `CREATE USER` = `CREATE ROLE WITH LOGIN`।

### Role তৈরি এবং Privilege দেওয়া

```sql
-- Application roles তৈরি করো (no login, শুধু privilege bundle)
CREATE ROLE app_readonly;
CREATE ROLE app_readwrite;
CREATE ROLE app_admin;

-- Roles এ privileges দাও
-- app_readonly
GRANT CONNECT ON DATABASE mydb TO app_readonly;
GRANT USAGE ON SCHEMA public TO app_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT ON TABLES TO app_readonly;
-- ALTER DEFAULT PRIVILEGES: ভবিষ্যতে তৈরি table এও এই privilege থাকবে

-- app_readwrite
GRANT CONNECT ON DATABASE mydb TO app_readwrite;
GRANT USAGE ON SCHEMA public TO app_readwrite;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_readwrite;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO app_readwrite;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_readwrite;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT USAGE, SELECT ON SEQUENCES TO app_readwrite;

-- Login users তৈরি করো এবং role assign করো
CREATE USER appuser   PASSWORD 'App@2024!' IN ROLE app_readwrite;
CREATE USER reporter  PASSWORD 'Report@2024!' IN ROLE app_readonly;
CREATE USER dbadmin   PASSWORD 'Admin@2024!' IN ROLE app_admin;

-- Specific table এ
GRANT SELECT ON orders TO app_readonly;
GRANT SELECT, INSERT ON orders TO app_readwrite;

-- Column level
GRANT SELECT (id, name, email) ON users TO reporter;
-- reporter শুধু id, name, email দেখতে পারবে (salary, password_hash না)

-- Revoke
REVOKE DELETE ON ALL TABLES IN SCHEMA public FROM app_readwrite;
REVOKE ALL PRIVILEGES ON ALL TABLES IN SCHEMA public FROM someuser;

-- Privileges দেখো
\du                          -- psql এ roles list
\dp users                    -- table এর privileges
SELECT grantee, privilege_type, table_name
FROM information_schema.role_table_grants
WHERE table_schema = 'public'
ORDER BY grantee, table_name;
```

### Schema-based Isolation (Multi-tenant)

```sql
-- প্রতিটা tenant এর আলাদা schema
CREATE SCHEMA tenant_abc;
CREATE SCHEMA tenant_xyz;

CREATE USER abc_user PASSWORD 'AbcPass@2024!';
GRANT CONNECT ON DATABASE mydb TO abc_user;
GRANT ALL ON SCHEMA tenant_abc TO abc_user;
GRANT ALL ON ALL TABLES IN SCHEMA tenant_abc TO abc_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA tenant_abc
    GRANT ALL ON TABLES TO abc_user;

-- abc_user শুধু tenant_abc schema দেখতে পারবে
```

---

## 9.3 Row Level Security (RLS)

MySQL তে নেই। PostgreSQL এর powerful feature। একই table এ বিভিন্ন user ভিন্ন rows দেখবে।

```sql
-- RLS enable করো
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Policy: প্রতিটা user শুধু নিজের orders দেখবে
CREATE POLICY user_isolation ON orders
    USING (user_id = (SELECT id FROM users WHERE username = current_user));

-- Multi-tenant: context variable দিয়ে
CREATE POLICY tenant_isolation ON orders
    USING (tenant_id = current_setting('app.tenant_id')::integer);

-- Application এ প্রতিটা request এ:
SET app.tenant_id = '42';
SELECT * FROM orders;  -- শুধু tenant_id=42 এর data আসবে

-- Superuser bypass করাও prevent করো
ALTER TABLE orders FORCE ROW LEVEL SECURITY;

-- Different policy for INSERT and SELECT
CREATE POLICY select_own ON orders FOR SELECT
    USING (user_id = current_setting('app.user_id')::integer);
CREATE POLICY insert_own ON orders FOR INSERT
    WITH CHECK (user_id = current_setting('app.user_id')::integer);

-- Policy দেখো
SELECT * FROM pg_policies WHERE tablename = 'orders';

-- RLS disable করো (testing)
ALTER TABLE orders DISABLE ROW LEVEL SECURITY;
```

---

## 9.4 Column Level Security

```sql
-- VIEW দিয়ে sensitive column hide করো
CREATE VIEW users_public AS
    SELECT id, username, email, full_name, created_at
    FROM users;
    -- password_hash, salary, ssn বাদ

-- reporter user কে শুধু view দিয়ে access দাও
GRANT SELECT ON users_public TO reporter;
REVOKE SELECT ON users FROM reporter;  -- Direct table access বন্ধ

-- Column GRANT
GRANT SELECT (id, name, email) ON users TO limited_user;
-- limited_user: SELECT id, name, email FROM users; ✅
-- limited_user: SELECT salary FROM users; ✅ ERROR
```

---

## 9.5 SSL/TLS

```ini
# postgresql.conf
ssl           = on
ssl_cert_file = 'server.crt'
ssl_key_file  = 'server.key'
ssl_ca_file   = 'root.crt'    # Client certificate verify করতে
ssl_ciphers   = 'HIGH:MEDIUM:+3DES:!aNULL'
ssl_min_protocol_version = 'TLSv1.2'
```

```bash
# Self-signed certificate (testing)
openssl req -new -x509 -days 365 -nodes \
    -out /var/lib/pgsql/16/data/server.crt \
    -keyout /var/lib/pgsql/16/data/server.key \
    -subj "/CN=pg-server"
chmod 600 /var/lib/pgsql/16/data/server.key
chown postgres:postgres /var/lib/pgsql/16/data/server.*

# Production: Let's Encrypt বা internal CA certificate ব্যবহার করো
```

```sql
-- SSL connection info দেখো
SELECT pid, ssl, version, cipher, bits
FROM pg_stat_ssl
WHERE pid = pg_backend_pid();

-- Force SSL for specific user
ALTER USER sensitive_user SET ssl = on;
-- pg_hba.conf এ hostssl rule দাও ওই user এর জন্য
```

---

## 9.6 pgcrypto — Database-Level Encryption

```sql
CREATE EXTENSION pgcrypto;

-- Symmetric encryption (AES)
INSERT INTO sensitive_data (ssn)
VALUES (pgp_sym_encrypt('123-45-6789', 'your_encryption_key'));

SELECT pgp_sym_decrypt(ssn::bytea, 'your_encryption_key') AS ssn
FROM sensitive_data;

-- Password hashing (bcrypt — industry standard)
-- Store করো:
INSERT INTO users (password_hash)
VALUES (crypt('userpassword', gen_salt('bf', 12)));
-- 'bf' = blowfish (bcrypt), 12 = work factor

-- Verify করো:
SELECT * FROM users
WHERE username = 'alice'
  AND password_hash = crypt('enteredpassword', password_hash);
-- এই query টা safe — timing attack proof

-- Random UUID
SELECT gen_random_uuid();

-- Random bytes (token generation)
SELECT encode(gen_random_bytes(32), 'hex');  -- 64-char hex token

-- Hash (SHA-256)
SELECT encode(digest('data_to_hash', 'sha256'), 'hex');
```

---

## 9.7 pgAudit — Audit Logging

কে কখন কী করেছে তার complete record রাখা। Compliance (PCI DSS, HIPAA, ISO 27001) এর জন্য।

```bash
# Install
sudo dnf install -y pgaudit16_16
# অথবা: sudo dnf install -y pgaudit
```

```ini
# postgresql.conf
shared_preload_libraries = 'pgaudit'

pgaudit.log = 'write, ddl'
# write: INSERT, UPDATE, DELETE, TRUNCATE
# ddl:   CREATE, DROP, ALTER
# read:  SELECT, COPY
# role:  GRANT, REVOKE, CREATE USER
# misc:  DISCARD, FETCH, CHECKPOINT
# all:   সব কিছু (production এ disk I/O বাড়ে)

pgaudit.log_catalog     = off   # System catalog queries log করো না (noise কমায়)
pgaudit.log_relation    = on    # প্রতিটা table/view আলাদা log entry
pgaudit.log_parameter   = on    # Query parameter values log করো
pgaudit.log_statement_once = off
```

```bash
sudo systemctl restart postgresql-16
```

```sql
-- Extension load করো
CREATE EXTENSION pgaudit;

-- Object audit (specific table)
-- postgresql.conf এ: pgaudit.log = 'none'  (global off)
-- Table level:
CREATE ROLE audit_role;
GRANT SELECT, INSERT, UPDATE, DELETE ON sensitive_table TO audit_role;
-- pgaudit.log_role = 'audit_role';  -- audit_role এর সব operation log হবে
```

```bash
# Audit log দেখো
sudo grep "AUDIT:" /var/lib/pgsql/16/data/log/postgresql-*.log | tail -20

# Format:
# AUDIT: SESSION,1,1,DDL,CREATE TABLE,,,"CREATE TABLE orders(...)",<none>
# AUDIT: SESSION,2,1,WRITE,INSERT,public,orders,"INSERT INTO orders VALUES(...)",<none>
# Type: SESSION (session audit) বা OBJECT (object audit)
```

---

## 9.8 Security Checklist

```
Authentication:
  ✅ pg_hba.conf এ trust method নেই (production এ)
  ✅ scram-sha-256 authentication (md5 এর চেয়ে secure)
  ✅ Remote postgres superuser access নেই
  ✅ SSL/TLS enabled (hostssl rule)
  ✅ idle_in_transaction_session_timeout set করা

Roles & Privileges:
  ✅ Least privilege — user কে minimum দরকারি privilege দাও
  ✅ Role-based access (সরাসরি user কে privilege না)
  ✅ ALTER DEFAULT PRIVILEGES দিয়ে ভবিষ্যত objects ও cover করা
  ✅ Column-level security sensitive data এ
  ✅ Row Level Security multi-tenant বা sensitive data এ

Data Security:
  ✅ Sensitive column pgcrypto দিয়ে encrypt
  ✅ Password hashing — bcrypt (crypt + gen_salt('bf'))
  ✅ pgBackRest দিয়ে backup encrypt করা

Audit:
  ✅ pgaudit installed এবং configured
  ✅ write এবং ddl operations log হচ্ছে
  ✅ Log rotation configured
  ✅ Log regularly review / SIEM এ ship করা

Network:
  ✅ listen_addresses specific IP এ (না হলে '*' + pg_hba.conf strict করো)
  ✅ Firewall port 5432 restrict
  ✅ pg_hba.conf এ specific CIDR blocks
  ✅ pgBouncer দিয়ে direct DB access বন্ধ করো (application → pgBouncer → PG)
```

---

# 10. High Availability & Replication

## 10.1 RPO এবং RTO — HA Design এর ভিত্তি

HA system design করার আগে business requirement জানতে হবে। RPO এবং RTO এই দুটো metric দিয়ে requirement define হয়।

```
RPO (Recovery Point Objective):
  "Disaster হলে কতটুকু data হারানো acceptable?"
  = সর্বশেষ কতক্ষণ আগের data থেকে restore করবো

  RPO = 0:     কোনো data loss নেই — Synchronous replication দরকার
  RPO = 5min:  5 মিনিটের data হারানো ok — WAL archive_timeout=5min
  RPO = 1hr:   1 ঘন্টার data হারানো ok — hourly backup যথেষ্ট

RTO (Recovery Time Objective):
  "Disaster হলে কতক্ষণের মধ্যে service ফিরিয়ে আনতে হবে?"
  = Downtime এর maximum acceptable duration

  RTO = 30s:   Patroni automatic failover দরকার
  RTO = 5min:  Manual promote (standby ready থাকলে)
  RTO = 1hr:   Backup থেকে restore acceptable
  RTO = 4hr:   Disaster recovery site থেকে restore
```

**RPO vs RTO — PostgreSQL Solution Mapping:**

| RPO | RTO | Solution |
|---|---|---|
| 0 | 30s | Synchronous Replication + Patroni |
| ~1s | 30s | Async Streaming + Patroni |
| 5min | 5min | Async Streaming + Manual failover |
| 1hr | 1hr | pgBackRest hourly backup + PITR |
| 24hr | 4hr | Daily backup + restore |

### Hands-on: তোমার Current RPO এবং RTO Measure করো

```sql
-- ─── Current RPO estimate ───
-- WAL archive কতক্ষণ পর পর হচ্ছে
SELECT
    last_archived_wal,
    last_archived_time,
    now() - last_archived_time AS archive_age,
    (SELECT setting FROM pg_settings WHERE name = 'archive_timeout') AS archive_timeout_setting
FROM pg_stat_archiver;
-- archive_age = তোমার current RPO (আরও ভালো করতে archive_timeout কমাও)

-- Replication lag = RPO for streaming replication
SELECT
    application_name,
    replay_lag,
    pg_size_pretty(sent_lsn - replay_lsn) AS lag_bytes
FROM pg_stat_replication;
-- replay_lag = failover এ potential data loss (async mode এ)
```

```bash
# ─── RTO Measurement — failover কতক্ষণে হয় সেটা test করো ───

# Test 1: Manual promote কতক্ষণ লাগে
# Primary stop করো:
time sudo systemctl stop postgresql-16

# Standby এ promote করো:
time psql -U postgres -c "SELECT pg_promote();"
# অথবা:
time sudo -u postgres /usr/pgsql-16/bin/pg_ctl promote -D /var/lib/pgsql/16/data

# Application reconnect হতে কতক্ষণ লাগে সেটাও add করো
# Total = আসল RTO

# Test 2: Patroni automatic failover কতক্ষণ লাগে
# Primary এর Patroni process kill করো:
time sudo systemctl stop patroni  # Primary এ
# অন্য terminal এ:
watch -n1 'patronictl -c /etc/patroni/patroni.yml list'
# কতক্ষণ এ Leader পরিবর্তন হয় সেটা দেখো এবং note করো

# ─── RTO document করো ───
echo "RTO Test Results: $(date)" >> /var/log/ha_test.log
echo "Manual promote: X seconds" >> /var/log/ha_test.log
echo "Patroni failover: Y seconds" >> /var/log/ha_test.log
```

---

## 10.2 Streaming Replication — কীভাবে কাজ করে

```
Primary Server:
  INSERT/UPDATE/DELETE হলে:
  ① Shared Buffers এ data page update (dirty)
  ② WAL record লেখো WAL Buffers এ
  ③ COMMIT → WAL flush to pg_wal/
  ④ WAL Sender process → Replica তে stream করো

Replica Server:
  WAL Receiver process:
  ① Primary এর WAL Sender এর সাথে connected
  ② WAL stream receive করে local pg_wal/ এ লেখে
  ③ Startup Process → WAL replay করে database update করে
  ④ Shared Buffers update হয়

Result:
  Replica = Primary এর exact copy (slight lag সহ)
  Replica তে read queries চলে (hot_standby=on)
  Primary তে write, Replica তে read → load distribution
```

```
Replication Types:
  Physical (Streaming):
    Block-level copy — exact byte-for-byte replica
    Same schema, same version required
    Read queries on Replica (hot_standby=on)

  Logical:
    Row-level changes replicate
    Different schema possible
    Different PG version possible (upgrade path)
    Selective table replication
```

---

**Hands-on: Replication Internal State দেখো**

```sql
-- ─── Primary এ: WAL Sender process দেখো ───
-- Replication চলার সময় pg_stat_activity তে WAL Sender দেখা যাবে
SELECT pid, usename, application_name, client_addr,
    backend_type, state, LEFT(query, 50) AS query
FROM pg_stat_activity
WHERE backend_type = 'walsender';
-- backend_type='walsender' → এটাই WAL Sender process

-- ─── Primary এ: WAL generation rate দেখো ───
SELECT pg_current_wal_lsn()          AS current_lsn,
    pg_walfile_name(pg_current_wal_lsn()) AS current_wal_file;
-- কিছু write করো, তারপর আবার দেখো → LSN বেড়েছে

-- ─── Primary এ: Replication stream details ───
SELECT
    application_name,
    client_addr,
    state,                    -- streaming / catchup / backup
    sent_lsn,                 -- Primary থেকে sent LSN
    write_lsn,                -- Replica disk এ write হয়েছে
    flush_lsn,                -- Replica fsync হয়েছে
    replay_lsn,               -- Replica apply হয়েছে
    sent_lsn - replay_lsn AS lag_lsn_diff,
    write_lag,                -- Network latency
    flush_lag,                -- Replica disk write lag
    replay_lag,               -- Replica apply lag
    sync_state
FROM pg_stat_replication;
-- state='streaming' এবং lag=0 = সব ঠিক আছে

-- ─── Primary এ: কতটা WAL জমে আছে pg_wal/ এ ───
SELECT count(*) AS wal_files,
    pg_size_pretty(sum(size)) AS total_wal_size
FROM pg_ls_waldir();
```

```bash
# ─── Replica এ: WAL Receiver process দেখো ───
psql -U postgres -c "
SELECT pid, status, receive_start_lsn, received_lsn,
    last_msg_send_time, last_msg_receipt_time,
    sender_host, sender_port
FROM pg_stat_wal_receiver;"
# status='streaming' এবং received_lsn বাড়ছে = ঠিক আছে

# ─── Real-time replication monitor করো ───
watch -n 2 'psql -U postgres -t -c "
SELECT
    CASE WHEN pg_is_in_recovery() THEN
        '"'"'STANDBY lag: '"'"'||(now()-pg_last_xact_replay_timestamp())::text
    ELSE
        '"'"'PRIMARY standbys: '"'"'||count(*)::text||'"'"' lag(max): '"'"'||
        COALESCE(pg_size_pretty(max(sent_lsn-replay_lsn)),'"'"'none'"'"')
    END
FROM pg_stat_replication;"'
# Ctrl+C দিয়ে বের হও

# ─── Live replication test করো ───
# Primary তে:
psql -U postgres -d mydb -c "
CREATE TABLE repl_test (id SERIAL, val TEXT, ts TIMESTAMPTZ DEFAULT NOW());
INSERT INTO repl_test (val) VALUES ('test-$(date +%s)');"

# Replica তে (কিছু সেকেন্ড পর):
psql -U postgres -d mydb -c "SELECT * FROM repl_test ORDER BY id DESC LIMIT 1;"
# Primary তে insert করা row দেখা গেলে ✅ replication কাজ করছে
```


## 10.3 Install থেকে Streaming Replication — End-to-End

### উভয় Server এ PostgreSQL Install

```bash
# Rocky Linux 9
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo dnf -qy module disable postgresql
sudo dnf install -y postgresql16-server postgresql16-contrib

# Primary তে Initialize করো
sudo /usr/pgsql-16/bin/postgresql-16-setup initdb
sudo systemctl start postgresql-16
sudo systemctl enable postgresql-16

# postgres user এর password set করো
sudo -u postgres psql -c "ALTER USER postgres PASSWORD 'PostgresPass@2024!';"
```

### Primary Server (172.16.93.140) Configure করো

```bash
sudo nano /var/lib/pgsql/16/data/postgresql.conf
```

```ini
listen_addresses         = '*'
wal_level                = replica
max_wal_senders          = 10
wal_keep_size            = 1GB
max_replication_slots    = 10
hot_standby              = on
archive_mode             = on
archive_command          = 'cp %p /archive/wal/%f'
```

```bash
# Archive directory
sudo mkdir -p /archive/wal
sudo chown postgres:postgres /archive/wal

# pg_hba.conf এ replication allow করো
sudo nano /var/lib/pgsql/16/data/pg_hba.conf
```

```
# Replication
host    replication     replicator      172.16.93.141/32        scram-sha-256
# সব connection (app, admin)
host    all             all             172.16.93.0/24          scram-sha-256
```

```sql
-- Replication user তৈরি করো
CREATE USER replicator REPLICATION LOGIN PASSWORD 'ReplicaPass@2024!';
```

```bash
# Firewall — Standby এর IP allow করো
sudo firewall-cmd --permanent --add-rich-rule='
  rule family=ipv4
  source address=172.16.93.141/32
  port port=5432 protocol=tcp accept'
sudo firewall-cmd --reload
sudo systemctl restart postgresql-16

# Verify
psql -c "SHOW wal_level;"                  # replica
psql -c "SHOW max_wal_senders;"            # 10
psql -c "SELECT pg_current_wal_lsn();"    # WAL position
```

### Standby Server (172.16.93.141) Setup করো

```bash
# Data directory clean করো
sudo systemctl stop postgresql-16
sudo rm -rf /var/lib/pgsql/16/data/*

# Primary থেকে base backup নাও
sudo -u postgres pg_basebackup \
    -h 172.16.93.140 \
    -U replicator \
    -D /var/lib/pgsql/16/data \
    -Xs \          # WAL stream করো
    -R \           # standby.signal + primary_conninfo auto-configure
    -P             # Progress দেখাও
# Password: ReplicaPass@2024!

# pg_basebackup automatically তৈরি করে:
# /var/lib/pgsql/16/data/standby.signal  ← এই file = standby mode
# /var/lib/pgsql/16/data/postgresql.auto.conf এ:
#   primary_conninfo = 'host=172.16.93.140 user=replicator ...'

# Standby postgresql.conf এ যোগ করো
sudo nano /var/lib/pgsql/16/data/postgresql.conf
```

```ini
hot_standby          = on      # Read queries allow
hot_standby_feedback = on      # Conflict prevention
```

```bash
sudo systemctl start postgresql-16
sudo systemctl enable postgresql-16

# Verify — Standby হিসেবে চলছে?
sudo -u postgres psql -c "SELECT pg_is_in_recovery();"
# t = standby (read-only mode)

# Lag দেখো
sudo -u postgres psql -c "SELECT now() - pg_last_xact_replay_timestamp() AS lag;"
```

### Primary তে Replication Verify করো

```sql
SELECT application_name, client_addr, state, sync_state,
    pg_size_pretty(sent_lsn - replay_lsn) AS lag
FROM pg_stat_replication;
-- state=streaming, lag=0 দেখালে সফল ✅
```

### Live Test

```sql
-- Primary তে
CREATE DATABASE repl_test;
\c repl_test
CREATE TABLE t1 (id SERIAL PRIMARY KEY, msg TEXT, created_at TIMESTAMPTZ DEFAULT NOW());
INSERT INTO t1 (msg) VALUES ('Replication working!');

-- Standby তে (কিছুক্ষণ পর)
\c repl_test
SELECT * FROM t1;
-- Row দেখা গেলে ✅ Replication সফল

-- Standby তে write test
INSERT INTO t1 (msg) VALUES ('should fail');
-- ERROR: cannot execute INSERT in a read-only transaction ✅
```

---

## 10.4 Replication Slots

Slot ছাড়া: Replica lag করলে Primary WAL delete করতে পারে → Replica sync হারায়।
Slot দিয়ে: Primary সেই Replica এর WAL retain করে যতক্ষণ না Replica সেটা পায়।

```sql
-- Physical replication slot তৈরি করো (Primary এ)
SELECT pg_create_physical_replication_slot('standby1_slot');

-- Standby তে use করো (postgresql.auto.conf এ)
-- primary_slot_name = 'standby1_slot'
-- অথবা: ALTER SYSTEM SET primary_slot_name = 'standby1_slot';

-- Slot status দেখো
SELECT
    slot_name,
    slot_type,
    active,
    restart_lsn,
    pg_size_pretty(
        pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)
    ) AS wal_retained
FROM pg_replication_slots;

-- ⚠️ WARNING: inactive slot WAL জমাতে থাকে → disk full হতে পারে!
-- Unused slot drop করো:
SELECT pg_drop_replication_slot('old_slot_name');

-- Monitoring: WAL retention 1GB+ হলে alert
```

---

## 10.5 Synchronous Replication

Default replication async — data loss possible (failover এ কিছু WAL miss হতে পারে)। Synchronous এ Primary COMMIT করার আগে Replica তে লেখা নিশ্চিত হয় → **RPO = 0**।

### Synchronous Commit Levels

```
synchronous_commit values:
  off          → WAL flush এর আগেই ack (fastest, local crash safe)
  local        → Local WAL flush হলে ack (replica নয়)
  remote_write → Replica memory তে পৌঁছালে ack (OS crash এ হারাতে পারে)
  on           → Replica WAL disk এ flush হলে ack
  remote_apply → Replica apply করলে ack (strictest, safest — RPO=0)
```

### Hands-on: Synchronous Replication Setup

```bash
# ─── Step 1: Standby এ application_name set করো ───
# Standby (172.16.93.141) এ postgresql.auto.conf দেখো
cat /var/lib/pgsql/16/data/postgresql.auto.conf
# primary_conninfo = 'host=172.16.93.140 user=replicator ...'

# application_name যোগ করো
psql -U postgres -c "
ALTER SYSTEM SET primary_conninfo = 'host=172.16.93.140 port=5432
    user=replicator password=ReplicaPass@2024!
    application_name=standby1';"
sudo systemctl restart postgresql-16
```

```bash
# ─── Step 2: Primary এ synchronous_standby_names set করো ───
# Primary (172.16.93.140) এ:
psql -U postgres -c "
ALTER SYSTEM SET synchronous_standby_names = 'standby1';
ALTER SYSTEM SET synchronous_commit = 'remote_apply';"
psql -U postgres -c "SELECT pg_reload_conf();"
```

```sql
-- ─── Step 3: Verify — sync_state দেখো ───
-- Primary এ:
SELECT application_name, client_addr, state, sync_state,
    write_lag, flush_lag, replay_lag
FROM pg_stat_replication;
-- sync_state = 'sync' হলে সফল ✅
-- sync_state = 'async' হলে application_name match করেনি

-- sync_state values:
-- sync      → এই standby synchronous (COMMIT এর আগে এর ack দরকার)
-- async     → asynchronous (ack দরকার নেই)
-- potential → sync হওয়ার candidate (sync down হলে এ হবে)
-- quorum    → quorum-based sync group এ
```

```bash
# ─── Step 4: Latency impact দেখো ───
# Async mode তে baseline নাও
psql -U postgres -c "ALTER SYSTEM SET synchronous_standby_names = '';"
psql -U postgres -c "SELECT pg_reload_conf();"
pgbench -U postgres -c 10 -T 30 mydb 2>&1 | grep tps

# Sync mode enable করো
psql -U postgres -c "ALTER SYSTEM SET synchronous_standby_names = 'standby1';"
psql -U postgres -c "ALTER SYSTEM SET synchronous_commit = 'remote_apply';"
psql -U postgres -c "SELECT pg_reload_conf();"
pgbench -U postgres -c 10 -T 30 mydb 2>&1 | grep tps
# TPS কমবে — network latency যোগ হয়েছে
```

```bash
# ─── Step 5: Standby down হলে Primary hang test ───
# Standby stop করো (172.16.93.141 এ):
sudo systemctl stop postgresql-16

# Primary তে INSERT চেষ্টা করো:
psql -U postgres -d mydb -c "INSERT INTO test (val) VALUES (1);" &
# Hang করবে! — synchronous standby নেই, Primary wait করছে

# Fix 1: ANY 1 দিয়ে — যেকোনো একটা standby থাকলেই হবে
# (Primary তে)
psql -U postgres -c "
ALTER SYSTEM SET synchronous_standby_names = 'ANY 1 (standby1, standby2)';"
psql -U postgres -c "SELECT pg_reload_conf();"

# Fix 2: Timeout set করো
psql -U postgres -c "
ALTER SYSTEM SET wal_receiver_timeout = '60s';"
# 60s পরে sync থেকে async এ fallback
```

**Trade-off Summary:**

| Mode | RPO | Write Latency | Standby Down হলে |
|---|---|---|---|
| `off` | Minutes | Fastest | No impact |
| `local` | Seconds | Fast | No impact |
| `remote_write` | Near-zero | Medium | No impact |
| `on` | Zero | Slow | Primary hangs |
| `remote_apply` | Zero | Slowest | Primary hangs |
| `ANY 1 (s1,s2)` | Zero | Slow | Other standby কাজ করে |

---

## 10.6 Logical Replication

Physical replication এ block-level copy হয় — same version, same schema দরকার। Logical replication এ row-level changes replicate হয় — different version, different schema, selective tables সম্ভব।

```
Physical vs Logical:
  Physical: WAL block copy → exact replica
  Logical:  "INSERT row X", "UPDATE row Y" → SQL-level changes

Logical Replication use cases:
  ① PG 14 → PG 16 zero-downtime version upgrade
  ② Specific tables এর subset replicate করো (reporting DB)
  ③ Different schema (column add/remove possible on subscriber)
  ④ Cross-database sync (same server, different DB)
  ⑤ Bidirectional (সাবধানে — conflict management দরকার)
```

### Hands-on: Logical Replication End-to-End

```bash
# ─── Setup: দুটো separate database use করবো ───
# Publisher: mydb (source)
# Subscriber: mydb_replica (destination — same বা different server এ হতে পারে)

# Step 1: Publisher এ wal_level = logical set করো
# (default replica → logical বাড়াতে হবে, restart দরকার)
psql -U postgres -c "SHOW wal_level;"
psql -U postgres -c "ALTER SYSTEM SET wal_level = logical;"
sudo systemctl restart postgresql-16
psql -U postgres -c "SHOW wal_level;"  -- logical confirm করো
```

```sql
-- ─── Step 2: Publisher DB তে Publication তৈরি করো ───
\c mydb

-- Specific tables
CREATE PUBLICATION mypub FOR TABLE users, orders, products;

-- অথবা সব tables
CREATE PUBLICATION mypub_all FOR ALL TABLES;

-- Publication দেখো
SELECT pubname, puballtables, pubinsert, pubupdate, pubdelete
FROM pg_publication;

-- কোন tables included দেখো
SELECT * FROM pg_publication_tables WHERE pubname = 'mypub';
```

```sql
-- ─── Step 3: Subscriber DB তে tables তৈরি করো ───
-- Subscriber এ same schema থাকতে হবে (কমপক্ষে replicated columns)
\c mydb_replica

CREATE TABLE users (
    id         BIGINT PRIMARY KEY,
    username   TEXT,
    email      TEXT,
    created_at TIMESTAMPTZ
);

CREATE TABLE orders (
    id         BIGINT PRIMARY KEY,
    user_id    INTEGER,
    total      NUMERIC(10,2),
    status     TEXT,
    created_at TIMESTAMPTZ
);

CREATE TABLE products (
    id    BIGINT PRIMARY KEY,
    name  TEXT,
    price NUMERIC(10,2)
);
```

```bash
# ─── Step 4: Publisher এর pg_hba.conf এ replication allow করো ───
# Subscriber এর IP allow করো:
sudo nano /var/lib/pgsql/16/data/pg_hba.conf
# যোগ করো:
# host mydb replicator 172.16.93.141/32 scram-sha-256
# (host + specific database — logical replication এ database specify করো)

sudo systemctl reload postgresql-16
```

```sql
-- ─── Step 5: Subscriber এ Subscription তৈরি করো ───
\c mydb_replica

CREATE SUBSCRIPTION mysub
    CONNECTION 'host=172.16.93.140 port=5432
                dbname=mydb
                user=replicator
                password=ReplicaPass@2024!'
    PUBLICATION mypub;

-- Initial data copy শুরু হবে (existing data sync)
-- শেষ হলে ongoing replication চালু হবে
```

```sql
-- ─── Step 6: Status verify করো ───

-- Subscriber এ:
SELECT subname, subenabled, subpublications,
       received_lsn, latest_end_lsn, latest_end_time
FROM pg_stat_subscription;
-- subenabled = t → subscription active

-- Publisher এ (Subscriber এর connection দেখবে):
SELECT application_name, client_addr, state, sent_lsn, replay_lsn
FROM pg_stat_replication
WHERE application_name LIKE 'mysub%';

-- Replication worker status:
SELECT * FROM pg_replication_slots WHERE slot_type = 'logical';
-- active = t → worker connected এবং replicating

-- ─── Step 7: Live test করো ───
-- Publisher এ insert করো:
\c mydb
INSERT INTO users (id, username, email, created_at)
VALUES (9999, 'testuser', 'test@example.com', NOW());

-- Subscriber এ confirm করো (কয়েক সেকেন্ড অপেক্ষা করো):
\c mydb_replica
SELECT * FROM users WHERE id = 9999;
-- Row দেখা গেলে ✅ Logical Replication কাজ করছে
```

### Logical Replication Management

```sql
-- Publication এ table যোগ বা বাদ দাও
ALTER PUBLICATION mypub ADD TABLE categories;
ALTER PUBLICATION mypub DROP TABLE products;
ALTER PUBLICATION mypub SET TABLE users, orders;  -- replace list

-- Subscription pause করো (maintenance)
ALTER SUBSCRIPTION mysub DISABLE;
ALTER SUBSCRIPTION mysub ENABLE;

-- Table manually resync করো (data diverge হলে)
ALTER SUBSCRIPTION mysub REFRESH PUBLICATION;  -- নতুন tables sync করো

-- Subscription drop করো
DROP SUBSCRIPTION mysub;
-- (replication slot Publisher এ automatically drop হয়)

-- Publication drop করো
DROP PUBLICATION mypub;
```

### PG Version Upgrade — Zero-Downtime (Logical Replication দিয়ে)

```
Scenario: PG 14 → PG 16 upgrade, downtime ছাড়া

Step 1: PG 16 নতুন server এ install করো
Step 2: PG 14 (old) → PG 16 (new) এ Logical Replication setup করো
Step 3: New server এ data sync হতে দাও (replica lag = 0)
Step 4: Application maintenance window এ:
         - Old server write বন্ধ করো (readonly mode)
         - New server এ lag = 0 confirm করো
         - Application new server এ point করো
         - Old server এ Publication drop করো
Step 5: Done — minimal downtime (seconds)
```

```bash
# Conflict handling — Subscriber এ conflict হলে
# (example: unique constraint violation)
# postgresql.conf on subscriber:
# log_min_messages = DEBUG1   -- conflict details দেখো

# Conflict skip করতে:
SELECT pg_replication_origin_advance(
    'pg_16395',          -- origin name (pg_stat_subscription থেকে)
    '0/1234567'::pg_lsn  -- skip করার LSN
);
```

---

## 10.7 Patroni + etcd + HAProxy — Production HA

Manual failover এ downtime হয়। Patroni দিয়ে automatic failover।

### Architecture

```
                 ┌──────────────────────────┐
                 │         HAProxy          │
                 │  Write: Port 5000        │
                 │  Read:  Port 5001        │
                 └────────────┬─────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│   PostgreSQL     │ │   PostgreSQL     │ │   PostgreSQL     │
│   Node 1         │ │   Node 2         │ │   Node 3         │
│   (Primary)      │ │   (Standby)      │ │   (Standby)      │
│   + Patroni      │ │   + Patroni      │ │   + Patroni      │
│   REST: 8008     │ │   REST: 8008     │ │   REST: 8008     │
└──────────────────┘ └──────────────────┘ └──────────────────┘
          │                   │                   │
          └───────────────────┼───────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│   etcd Node 1    │ │   etcd Node 2    │ │   etcd Node 3    │
│   Paxos Consensus│                                          │
└──────────────────────────────────────────────────────────────┘

Component কাজ:
  Patroni: PostgreSQL HA agent
           - Primary elected কে promote করে
           - Standbys কে নতুন Primary follow করায়
           - Health check করে
           - REST API expose করে (HAProxy এটা check করে)

  etcd:    Distributed key-value store
           - Leader lock রাখে
           - কে Primary সেটা store করে
           - Consensus (majority agree করলে) failover হয়

  HAProxy: TCP load balancer
           - Patroni REST API check করে Primary identify করে
           - Write traffic → Primary (port 5000)
           - Read traffic → Any node (port 5001)
```

### etcd Cluster Setup (সব তিনটা node এ)

```bash
sudo dnf install -y etcd

# Node 1 (172.16.93.140) এ /etc/etcd/etcd.conf.yml:
```

```yaml
name: etcd1
data-dir: /var/lib/etcd
listen-client-urls: http://172.16.93.140:2379,http://127.0.0.1:2379
advertise-client-urls: http://172.16.93.140:2379
listen-peer-urls: http://172.16.93.140:2380
initial-advertise-peer-urls: http://172.16.93.140:2380
initial-cluster: >
  etcd1=http://172.16.93.140:2380,
  etcd2=http://172.16.93.141:2380,
  etcd3=http://172.16.93.142:2380
initial-cluster-token: pg-ha-cluster
initial-cluster-state: new
```

```bash
sudo systemctl start etcd && sudo systemctl enable etcd

# Cluster health check
etcdctl --endpoints=http://172.16.93.140:2379 endpoint health
etcdctl --endpoints=http://172.16.93.140:2379,http://172.16.93.141:2379,http://172.16.93.142:2379 member list
```

### Patroni Install ও Configure

```bash
# সব PostgreSQL node এ
sudo pip3 install patroni[etcd] psycopg2-binary

sudo mkdir -p /etc/patroni
sudo nano /etc/patroni/patroni.yml  # Node 1 এর config
```

```yaml
scope: pg-cluster
namespace: /service/
name: pg-node1                   # প্রতিটা node এ আলাদা name

restapi:
  listen: 172.16.93.140:8008     # Patroni REST API (HAProxy এটা check করে)
  connect_address: 172.16.93.140:8008

etcd:
  hosts: 172.16.93.140:2379,172.16.93.141:2379,172.16.93.142:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576    # 1MB — এর বেশি lag থাকলে failover candidate না
    postgresql:
      use_pg_rewind: true               # pg_rewind দিয়ে old primary rejoin করবে
      use_slots: true
      parameters:
        max_connections: 200
        shared_buffers: 4GB
        wal_level: replica
        hot_standby: "on"
        max_wal_senders: 10
        max_replication_slots: 10
        wal_keep_size: 1GB
        archive_mode: "on"
        archive_command: "cp %p /archive/wal/%f"

  initdb:
    - encoding: UTF8
    - data-checksums              # Corruption detection এর জন্য

  pg_hba:                         # pg_hba.conf automatically manage করবে
    - host replication replicator 0.0.0.0/0 scram-sha-256
    - host all all 0.0.0.0/0 scram-sha-256

  users:
    admin:
      password: Admin@2024!
      options: [createdb, createrole]

postgresql:
  listen: 172.16.93.140:5432
  connect_address: 172.16.93.140:5432
  data_dir: /var/lib/pgsql/16/data
  bin_dir: /usr/pgsql-16/bin
  pgpass: /tmp/pgpass               # Password file

  authentication:
    replication:
      username: replicator
      password: ReplicaPass@2024!
    superuser:
      username: postgres
      password: PostgresPass@2024!

tags:
  nofailover: false
  noloadbalance: false
  clonedfrom: false
  nosync: false
```

```bash
# Patroni systemd service
sudo tee /etc/systemd/system/patroni.service << 'EOF'
[Unit]
Description=Patroni PostgreSQL HA
After=syslog.target network.target

[Service]
Type=simple
User=postgres
Group=postgres
ExecStart=/usr/local/bin/patroni /etc/patroni/patroni.yml
KillMode=process
TimeoutSec=30
Restart=no

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl start patroni && sudo systemctl enable patroni

# Cluster status দেখো
patronictl -c /etc/patroni/patroni.yml list
```

```
# Expected output:
+ Cluster: pg-cluster ────────────+----+-----------+
| Member   | Host              | Role   | State   | TL | Lag in MB |
+──────────+───────────────────+────────+─────────+────+───────────+
| pg-node1 | 172.16.93.140:5432| Leader | running |  1 |           |
| pg-node2 | 172.16.93.141:5432| Replica| running |  1 |         0 |
| pg-node3 | 172.16.93.142:5432| Replica| running |  1 |         0 |
+──────────+───────────────────+────────+─────────+────+───────────+
```

### HAProxy Configure করো

```bash
sudo dnf install -y haproxy
sudo nano /etc/haproxy/haproxy.cfg
```

```ini
global
    maxconn 1000
    log /dev/log local0

defaults
    mode tcp
    log global
    retries 2
    timeout client  30m
    timeout connect 4s
    timeout server  30m
    option clitcpka

# HAProxy Stats page
listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /
    stats refresh 5s

# Write traffic → Primary ONLY (Port 5000)
frontend pg_write
    bind *:5000
    default_backend pg_primary

backend pg_primary
    option httpchk GET /master     # Patroni REST API: /master = Primary면 200, Standby면 503
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server pg1 172.16.93.140:5432 maxconn 100 check port 8008
    server pg2 172.16.93.141:5432 maxconn 100 check port 8008
    server pg3 172.16.93.142:5432 maxconn 100 check port 8008

# Read traffic → Any healthy node (Port 5001)
frontend pg_read
    bind *:5001
    default_backend pg_replicas

backend pg_replicas
    option httpchk GET /health     # /health = সব healthy node 200 return করে
    http-check expect status 200
    balance roundrobin
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server pg1 172.16.93.140:5432 maxconn 100 check port 8008
    server pg2 172.16.93.141:5432 maxconn 100 check port 8008
    server pg3 172.16.93.142:5432 maxconn 100 check port 8008
```

```bash
sudo systemctl start haproxy && sudo systemctl enable haproxy

# Test
psql -h haproxy_ip -p 5000 -U postgres -c "SELECT inet_server_addr();"
# Primary IP দেখাবে

psql -h haproxy_ip -p 5001 -U postgres -c "SELECT pg_is_in_recovery();"
# t = connected to Standby (read node)

# Application config:
# Write: postgresql://haproxy_ip:5000/mydb
# Read:  postgresql://haproxy_ip:5001/mydb
```

### Patroni Operations

```bash
# Cluster status
patronictl -c /etc/patroni/patroni.yml list

# Planned switchover (graceful, no downtime)
patronictl -c /etc/patroni/patroni.yml switchover pg-cluster \
    --master pg-node1 --candidate pg-node2 --force

# Manual failover (emergency)
patronictl -c /etc/patroni/patroni.yml failover pg-cluster --force

# Node reinitialize (sync হারানো node কে re-clone করো)
patronictl -c /etc/patroni/patroni.yml reinit pg-cluster pg-node2

# Cluster-wide config change (সব node এ apply হয়)
patronictl -c /etc/patroni/patroni.yml edit-config

# Patroni pause করো (maintenance — auto-failover বন্ধ)
patronictl -c /etc/patroni/patroni.yml pause pg-cluster
patronictl -c /etc/patroni/patroni.yml resume pg-cluster

# History দেখো
patronictl -c /etc/patroni/patroni.yml history pg-cluster
```

---

## 10.8 pgBouncer — Connection Pooler

**কেন দরকার?**
PostgreSQL এ প্রতিটা connection = ~5-10MB shared memory + OS process। 1000 connection = 5-10GB শুধু connections এ। pgBouncer connection reuse করে।

```
Without pgBouncer:
  App (500 connections) → PostgreSQL (500 processes, 2-5GB)

With pgBouncer:
  App (500 connections) → pgBouncer → PostgreSQL (20 connections, 200MB)
  App ভাবছে 500 connection আছে
  PostgreSQL এ আসলে 20টা
```

```bash
sudo dnf install -y pgbouncer
sudo nano /etc/pgbouncer/pgbouncer.ini
```

```ini
[databases]
mydb = host=127.0.0.1 port=5432 dbname=mydb

# Patroni/HAProxy এর সাথে
mydb = host=haproxy_ip port=5000 dbname=mydb

[pgbouncer]
listen_addr        = *
listen_port        = 6432
auth_type          = scram-sha-256
auth_file          = /etc/pgbouncer/userlist.txt

# Pool Mode — সবচেয়ে গুরুত্বপূর্ণ setting
pool_mode          = transaction
# session:     Client এর পুরো session এ একটা backend connection hold করে
#              MySQL Thread Cache এর মতো
#              SET, LISTEN safe, কিন্তু concurrency কম
# transaction: Transaction এর সময়ই backend connection hold করে
#              1000 client, 20 backend connection → efficient
#              SET commands (session level) কাজ করে না
# statement:   প্রতিটা statement এ release করে
#              autocommit only, খুব rare use case

default_pool_size  = 20       # Per database per user কতটা backend connection
max_client_conn    = 1000     # pgBouncer তে maximum client connections
reserve_pool_size  = 5        # Emergency reserve pool
server_idle_timeout = 600

logfile = /var/log/pgbouncer/pgbouncer.log
pidfile = /var/run/pgbouncer/pgbouncer.pid
admin_users = pgbouncer_admin
```

```bash
# userlist.txt তৈরি করো
# PostgreSQL থেকে hash নাও:
sudo -u postgres psql -t -c \
    "SELECT '\"' || usename || '\" \"' || passwd || '\"' FROM pg_shadow WHERE usename = 'appuser';"
# Output: "appuser" "SCRAM-SHA-256$..."
echo '"appuser" "SCRAM-SHA-256$..."' | sudo tee /etc/pgbouncer/userlist.txt
echo '"pgbouncer_admin" "admin_password"' | sudo tee -a /etc/pgbouncer/userlist.txt

sudo systemctl start pgbouncer && sudo systemctl enable pgbouncer

# Application এখন port 6432 তে connect করবে (5432 এর বদলে)
psql -h localhost -p 6432 -U appuser -d mydb

# pgBouncer admin console
psql -U pgbouncer_admin -p 6432 pgbouncer
SHOW STATS;      # Statistics
SHOW POOLS;      # Pool status
SHOW CLIENTS;    # Connected clients
SHOW SERVERS;    # Backend connections
RELOAD;          # Config reload
```

---

## 10.9 pgPool-II

pgBouncer এর চেয়ে বেশি features — connection pooling + read/write split + query routing।

```bash
sudo dnf install -y pgpool-II-pg16
sudo nano /etc/pgpool-II/pgpool.conf
```

```ini
listen_addresses = '*'
port = 9999

# Backends
backend_hostname0 = '172.16.93.140'   # Primary
backend_port0     = 5432
backend_weight0   = 1
backend_flag0     = 'ALWAYS_PRIMARY'

backend_hostname1 = '172.16.93.141'   # Standby
backend_port1     = 5432
backend_weight1   = 1
backend_flag1     = 'DISALLOW_TO_FAILOVER'   # Patroni failover করবে, pgPool না

# Connection pool
num_init_children = 32
max_pool          = 4

# Load balancing
load_balance_mode = on    # Read queries Standbys এ যাবে
```

**pgBouncer vs pgPool-II:**
| | pgBouncer | pgPool-II |
|---|---|---|
| Primary use | Connection pooling | Pooling + more |
| Read/write split | No (external HAProxy দরকার) | Yes, built-in |
| Complexity | Low | High |
| Performance overhead | Very low | Higher |
| Production choice | Most teams prefer | When R/W split built-in চাই |
| With Patroni | pgBouncer + HAProxy | pgPool-II alone |

---

## 10.10 Failover

### Manual Failover

```bash
# Primary down হলে Standby কে Primary করো

# Option 1: pg_ctl promote
sudo -u postgres /usr/pgsql-16/bin/pg_ctl promote \
    -D /var/lib/pgsql/16/data

# Option 2: SQL function (PostgreSQL 12+)
sudo -u postgres psql -c "SELECT pg_promote();"

# Option 3: Trigger file
sudo -u postgres touch /tmp/promote_trigger
# postgresql.conf: promote_trigger_file = '/tmp/promote_trigger'

# Verify
sudo -u postgres psql -c "SELECT pg_is_in_recovery();"
# f = Primary হয়ে গেছে ✅
```

### Automatic Failover (Patroni)

```
Primary Node fails:
  1. Patroni: heartbeat miss → etcd এর leader lock expire
  2. etcd: leader lock free হয়ে যায়
  3. Remaining Patroni agents: leader election শুরু
  4. Most up-to-date Standby (lowest lag): leader lock পায়
  5. Winner: pg_ctl promote → Primary
  6. Other Standbys: নতুন Primary এর সাথে replication শুরু
  7. HAProxy: /master check → নতুন Primary কে detect করে
  8. Traffic: নতুন Primary তে route হয়

Total time: ~30-60 seconds
```

### pg_rewind — Failed Primary কে Standby হিসেবে Rejoin

```bash
# Scenario: Node1 ছিল Primary, failover এ Node2 Primary হয়েছে
# Node1 restore হয়েছে — Standby হিসেবে rejoin করতে চাই

# Node1 stop করো
sudo systemctl stop postgresql-16

# pg_rewind চালাও (Node2 = new Primary)
sudo -u postgres /usr/pgsql-16/bin/pg_rewind \
    --target-pgdata=/var/lib/pgsql/16/data \
    --source-server='host=172.16.93.141 user=postgres password=xxx'
# pg_rewind: Node1 এর diverged WAL সরিয়ে Node2 এর WAL আনে

# Standby signal এবং config
sudo -u postgres touch /var/lib/pgsql/16/data/standby.signal
# postgresql.auto.conf এ primary_conninfo আপডেট করো:
sudo -u postgres psql -c "ALTER SYSTEM SET primary_conninfo = 'host=172.16.93.141 user=replicator password=xxx';"

sudo systemctl start postgresql-16
# Node1 এখন Node2 এর Standby হিসেবে চলবে

# Patroni দিয়ে এটা automatic:
patronictl -c /etc/patroni/patroni.yml reinit pg-cluster pg-node1
```

---

## 10.11 Replication Topology Comparison

| Topology | Complexity | Failover | Data Loss | Use Case |
|---|---|---|---|---|
| Primary + Standby | Simple | Manual (5-30min) | Possible | Small app |
| Primary + Multi-Standby | Medium | Manual | Possible | Read scaling |
| Synchronous Replication | Medium | Manual, zero loss | None | Financial |
| Patroni + etcd + HAProxy | High | Auto (30-60s) | Possible | Production HA |
| Synchronous + Patroni | High | Auto, zero loss | None | Critical production |
| Logical Replication | Medium | N/A | N/A | Migration, filtering |
| Delayed Standby | Low | Manual | N/A | Accidental delete protection |
| Cascading Replication | Medium | Complex | Possible | Multi-datacenter |

**Hands-on: তোমার Current Topology Identify করো**

```sql
-- ─── এই server কোন role এ আছে? ───
SELECT
    CASE
        WHEN pg_is_in_recovery() = false THEN 'PRIMARY — writes allowed'
        WHEN pg_is_in_recovery() = true  THEN 'STANDBY — read only'
    END AS current_role,
    inet_server_addr()  AS server_ip,
    version()           AS pg_version;

-- ─── Primary হলে: কতটা Standby connected? ───
SELECT
    COUNT(*)                                          AS standby_count,
    SUM(CASE WHEN sync_state='sync'  THEN 1 ELSE 0 END) AS sync_standbys,
    SUM(CASE WHEN sync_state='async' THEN 1 ELSE 0 END) AS async_standbys,
    pg_size_pretty(MAX(sent_lsn - replay_lsn))        AS max_lag
FROM pg_stat_replication;

-- ─── Standby হলে: কোন Primary তে connected? ───
SELECT
    pg_last_wal_receive_lsn()                                   AS received_lsn,
    pg_last_wal_replay_lsn()                                    AS replayed_lsn,
    now() - pg_last_xact_replay_timestamp()                     AS lag_time,
    pg_size_pretty(pg_wal_lsn_diff(
        pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn()))   AS lag_bytes
WHERE pg_is_in_recovery();

-- ─── Replication slots কী আছে? ───
SELECT slot_name, slot_type, active,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS wal_retained
FROM pg_replication_slots;

-- ─── Synchronous standby names কী set আছে? ───
SELECT name, setting FROM pg_settings
WHERE name IN ('synchronous_standby_names','synchronous_commit','wal_level');
```

```bash
# ─── Complete topology summary script ───
cat > /usr/local/bin/pg_topology.sh << 'SCRIPT'
#!/bin/bash
PSQL="psql -U postgres -t -A -c"

echo "============================================"
echo " PostgreSQL Topology Report - $(date)"
echo "============================================"

echo ""
echo "--- THIS SERVER ---"
IS_STANDBY=$($PSQL "SELECT pg_is_in_recovery();")
if [ "$IS_STANDBY" = "f" ]; then
    echo "Role: PRIMARY"
    echo ""
    echo "--- CONNECTED STANDBYS ---"
    $PSQL "SELECT application_name||' | '||client_addr||' | sync:'||sync_state||' | lag:'||COALESCE(replay_lag::text,'N/A') FROM pg_stat_replication;"
    STANDBY_COUNT=$($PSQL "SELECT count(*) FROM pg_stat_replication;")
    echo "Total standbys: $STANDBY_COUNT"
else
    echo "Role: STANDBY (read-only)"
    echo ""
    echo "--- REPLICATION STATUS ---"
    $PSQL "SELECT 'Connected to primary: '||COALESCE(sender_host,'unknown')||':'||COALESCE(sender_port::text,'?') FROM pg_stat_wal_receiver;"
    $PSQL "SELECT 'Lag time: '||(now() - pg_last_xact_replay_timestamp())::text FROM pg_stat_activity LIMIT 1;"
    $PSQL "SELECT 'Is recovery paused: '||pg_is_wal_replay_paused()::text;"
fi

echo ""
echo "--- REPLICATION SLOTS ---"
$PSQL "SELECT slot_name||' | '||slot_type||' | active:'||active::text||' | WAL retained: '||pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(),COALESCE(restart_lsn,'0/0'::pg_lsn))) FROM pg_replication_slots;" 2>/dev/null || echo "No slots"

echo ""
echo "--- WAL SETTINGS ---"
$PSQL "SELECT name||': '||setting FROM pg_settings WHERE name IN ('wal_level','synchronous_standby_names','synchronous_commit') ORDER BY name;"
echo "============================================"
SCRIPT
chmod +x /usr/local/bin/pg_topology.sh
sudo -u postgres /usr/local/bin/pg_topology.sh
```

---

## 10.12 Troubleshooting Replication

### Standby Connect হচ্ছে না

```bash
# pg_hba.conf check
grep replication /var/lib/pgsql/16/data/pg_hba.conf

# Primary তে connection test
psql -h 172.16.93.140 -U replicator -d replication -c "SELECT 1;"
# -d replication: replication pseudo-database

# Error log
sudo tail -50 /var/lib/pgsql/16/data/log/postgresql-*.log
```

| Error | Cause | Solution |
|---|---|---|
| `no pg_hba.conf entry for replication` | pg_hba.conf এ line নেই | Add করো, reload |
| `password authentication failed` | Wrong password | User recreate করো |
| `connection refused` | Firewall বা port বন্ধ | Firewall check, port open করো |
| `WAL segment ... has already been removed` | WAL deleted before Replica got it | `wal_keep_size` বাড়াও বা Replication Slot ব্যবহার করো |
| `requested starting point is ahead` | Replica এর WAL position Primary তে নেই | pg_basebackup দিয়ে re-clone করো |

### Replica Lag বেশি

```sql
-- Primary তে
SELECT application_name,
    pg_size_pretty(sent_lsn - replay_lsn) AS lag,
    replay_lag
FROM pg_stat_replication;

-- Cause:
-- Network bandwidth কম?  → wal_compression = on
-- Replica CPU বেশি busy? → Replica এর load check করো
-- Long-running query on Replica blocking apply?
--   → hot_standby_feedback = on করো Primary তে

-- Replica এ
SELECT now() - pg_last_xact_replay_timestamp() AS lag;
```

---

## 10.13 Quick Reference Commands

**Installation (Rocky Linux 9):**

```bash
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo dnf -qy module disable postgresql
sudo dnf install -y postgresql16-server postgresql16-contrib
sudo /usr/pgsql-16/bin/postgresql-16-setup initdb
sudo systemctl start postgresql-16 && sudo systemctl enable postgresql-16
```

**psql Essential Commands:**

| Command | কাজ |
|---|---|
| `\l` বা `\list` | সব database list |
| `\c dbname` | Database connect |
| `\dt` | Current schema এর tables |
| `\dt *.*` | সব schema এর tables |
| `\d tablename` | Table structure + indexes |
| `\di` | Indexes list |
| `\dv` | Views list |
| `\df` | Functions list |
| `\du` | Users/Roles list |
| `\dp tablename` | Table privileges |
| `\dn` | Schemas list |
| `\x` | Expanded display toggle |
| `\timing` | Query timing toggle |
| `\i file.sql` | SQL file execute করো |
| `\copy table TO/FROM file` | File import/export |
| `\! ls` | Shell command |
| `\q` | Quit |
| `\?` | psql help |
| `\h CREATE TABLE` | SQL command help |

**Primary Server:**

| Query | কাজ |
|---|---|
| `SELECT pg_current_wal_lsn();` | Current WAL position |
| `SELECT * FROM pg_stat_replication;` | Replica status |
| `SELECT * FROM pg_replication_slots;` | Replication slots |
| `SELECT pg_switch_wal();` | Force WAL switch |
| `SELECT pg_create_physical_replication_slot('name');` | Create slot |
| `SELECT pg_drop_replication_slot('name');` | Drop slot |
| `CHECKPOINT;` | Force checkpoint |

**Standby Server:**

| Query | কাজ |
|---|---|
| `SELECT pg_is_in_recovery();` | Standby? (t=yes, f=primary) |
| `SELECT pg_last_wal_receive_lsn();` | Last received LSN |
| `SELECT pg_last_wal_replay_lsn();` | Last applied LSN |
| `SELECT now() - pg_last_xact_replay_timestamp();` | Lag time |
| `SELECT pg_promote();` | Promote to Primary |
| `SELECT pg_wal_replay_resume();` | Resume recovery |

**Both:**

| Query | কাজ |
|---|---|
| `SHOW data_directory;` | Data directory |
| `SHOW config_file;` | Config file path |
| `SELECT pg_reload_conf();` | Config reload |
| `EXPLAIN (ANALYZE, BUFFERS) query;` | Execution plan |
| `VACUUM VERBOSE ANALYZE table;` | Vacuum + analyze |
| `SELECT pg_cancel_backend(pid);` | Cancel query |
| `SELECT pg_terminate_backend(pid);` | Kill connection |
| `SELECT txid_current();` | Current XID |
| `SELECT * FROM pg_stat_activity;` | All connections |

**Patroni:**

| Command | কাজ |
|---|---|
| `patronictl list` | Cluster status |
| `patronictl failover cluster` | Failover |
| `patronictl switchover cluster` | Planned switchover |
| `patronictl reinit cluster node` | Node re-clone |
| `patronictl edit-config` | Cluster config edit |
| `patronictl pause cluster` | Pause auto-failover |
| `patronictl resume cluster` | Resume |
| `patronictl history cluster` | Failover history |

---

## 10.14 Delayed Standby — Accidental Data Loss Protection

Delayed standby হলো এমন একটা Replica যেটা ইচ্ছাকৃতভাবে N মিনিট/ঘন্টা পিছিয়ে থাকে। কেউ ভুলে `DELETE FROM orders WHERE 1=1` করলে সেই change এখনো delayed standby তে apply হয়নি।

```
Normal replication:
  Primary → WAL → Standby (lag: milliseconds)

Delayed standby:
  Primary → WAL → Delayed Standby (lag: 1 hour)
  Primary এ ভুল DELETE → 1 ঘন্টা পরে standby তে apply হবে
  → সেই 1 ঘন্টার মধ্যে promote করলে data পাওয়া যাবে
```

```ini
# Delayed standby এর postgresql.conf তে:
recovery_min_apply_delay = '1h'
# অথবা: '30min', '2h', '6h'

hot_standby = on   # Read queries allow (optional, delay এ read করলে পুরনো data দেখাবে)
```

```bash
# Setup (pg_basebackup দিয়ে standby তৈরি করার পরে)
# Delayed standby এর postgresql.auto.conf এ:
# primary_conninfo = 'host=172.16.93.140 user=replicator password=...'
# recovery_min_apply_delay = '1h'
# Restart করো
sudo systemctl restart postgresql-16

# Verify
psql -c "SELECT now() - pg_last_xact_replay_timestamp() AS delay;"
# ~1h দেখাবে
```

**Emergency recover from delayed standby:**
```bash
# Production এ disaster হলে:

# Step 1: Delay bypass করো (apply শুরু করো)
psql -c "SELECT pg_wal_replay_resume();"
# delay থাকলেও এই মুহূর্তে apply শুরু হয়

# Step 2: Target time এ stop করতে চাইলে
# postgresql.conf এ:
# recovery_target_time = '2024-03-08 14:29:59+06'  ← disaster এর ঠিক আগে
# recovery_target_action = 'promote'
# Restart করো

# Step 3: Promote করো
psql -c "SELECT pg_promote();"
```

---

## 10.15 Cascading Replication

Primary → Standby1 → Standby2 (chain)। Standby2 Primary থেকে সরাসরি না নিয়ে Standby1 থেকে WAL নেয়।

```
Primary ─── WAL Sender ──► Standby1 (Intermediate)
                                │
                                └── WAL Sender ──► Standby2 (Cascade)

কখন ব্যবহার করবো:
  Different datacenter এ replica → WAN bandwidth save
  Primary এর max_wal_senders limit পূরণ
  "Hub" standby — একটা intermediate standby থেকে অনেক replica
```

```bash
# Standby1 (Intermediate) এ:
# postgresql.conf:
# wal_level = replica      (standby ও WAL send করতে পারবে)
# max_wal_senders = 5       (standby থেকে আরো replica)
# hot_standby = on

# Standby2 এর postgresql.auto.conf:
# primary_conninfo = 'host=172.16.93.141 user=replicator password=...'
#                          ↑ Standby1 এর IP, Primary এর না!

# Verify: Standby2 থেকে কোথায় connected
SELECT client_addr FROM pg_stat_replication;  -- Standby1 এ চালাও
-- Standby2 এর IP দেখাবে
```

---

## 10.16 Read Scaling Architecture

Read traffic distribute করে Primary এর load কমানো।

```
Architecture options:

Option 1: HAProxy + Multiple Standbys (সহজ)
  App → HAProxy:5001 (read) → roundrobin → Standby1, Standby2, Standby3
  App → HAProxy:5000 (write) → Primary

  Pros: Simple, HAProxy already আছে
  Cons: Any standby might have slight lag

Option 2: pgBouncer + Read Replica routing
  Write connection: host=haproxy:5000
  Read connection:  host=haproxy:5001

Option 3: Application-level routing
  DB_WRITE = "postgresql://primary:5432/mydb"
  DB_READ  = "postgresql://haproxy:5001/mydb"
  Write ORM → DB_WRITE
  Read ORM  → DB_READ (Django: using('replica'))
```

```python
# Django read replica setup
# settings.py:
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'HOST': 'haproxy-ip',
        'PORT': '5000',  # Write port
        'NAME': 'mydb',
    },
    'replica': {
        'ENGINE': 'django.db.backends.postgresql',
        'HOST': 'haproxy-ip',
        'PORT': '5001',  # Read port
        'NAME': 'mydb',
    }
}

DATABASE_ROUTERS = ['myapp.routers.ReadReplicaRouter']

# routers.py:
class ReadReplicaRouter:
    def db_for_read(self, model, **hints):
        return 'replica'  # সব read → replica

    def db_for_write(self, model, **hints):
        return 'default'  # সব write → primary

    def allow_relation(self, obj1, obj2, **hints):
        return True
```

```sql
-- Standby তে read load দেখো
SELECT count(*) FROM pg_stat_activity WHERE state = 'active';
SELECT query_start, LEFT(query, 80) FROM pg_stat_activity
WHERE state = 'active' AND pg_is_in_recovery();
```

---

## 10.17 HA Troubleshooting Scenarios

### Scenario 1: Replica সরাসরি Connect হচ্ছে না

```bash
# Error: "could not connect to the primary server"
# Step 1: Primary তে test করো
psql -h 172.16.93.140 -U replicator -d replication -c "SELECT 1;"
# Error → pg_hba.conf এ entry নেই বা wrong password

# Step 2: pg_hba.conf check করো Primary তে
grep replication /var/lib/pgsql/16/data/pg_hba.conf
# host replication replicator 172.16.93.141/32 scram-sha-256 থাকতে হবে

# Step 3: Firewall check
sudo firewall-cmd --list-all | grep 5432
# Port open না থাকলে:
sudo firewall-cmd --permanent --add-port=5432/tcp
sudo firewall-cmd --reload

# Step 4: Replica এর postgresql.auto.conf দেখো
cat /var/lib/pgsql/16/data/postgresql.auto.conf
# primary_conninfo = 'host=... user=... password=...'
# Password ঠিক আছে?
```

### Scenario 2: Replica Lag বাড়ছে

```sql
-- Primary তে lag দেখো
SELECT application_name,
    pg_size_pretty(sent_lsn - replay_lsn) AS lag_size,
    write_lag, flush_lag, replay_lag
FROM pg_stat_replication;

-- Diagnosis:
-- write_lag বেশি  → Network bandwidth কম বা Replica disk slow
-- flush_lag বেশি  → Replica disk I/O slow
-- replay_lag বেশি → Replica CPU high, বা query conflict (hot_standby_feedback=off)

-- Query conflict দেখো (Replica তে)
SELECT COUNT(*) FROM pg_stat_activity WHERE wait_event_type = 'Recovery';
-- এরা query conflict এর কারণে wait করছে

-- Fix: Primary তে
ALTER SYSTEM SET hot_standby_feedback = on;
SELECT pg_reload_conf();
```

### Scenario 3: Patroni Cluster Primary নেই (No Leader)

```bash
patronictl -c /etc/patroni/patroni.yml list
# সব node: "Replica" বা "stopped"

# Step 1: etcd healthy?
etcdctl --endpoints=http://172.16.93.140:2379,http://172.16.93.141:2379,http://172.16.93.142:2379 endpoint health
# majority (2 of 3) healthy হতে হবে

# Step 2: Patroni logs দেখো
sudo journalctl -u patroni -n 50 --no-pager

# Step 3: Manual leader force করো
patronictl -c /etc/patroni/patroni.yml failover pg-cluster --master pg-node1 --force

# Step 4: Node reinitialize করো যদি দরকার হয়
patronictl -c /etc/patroni/patroni.yml reinit pg-cluster pg-node2
```

### Scenario 4: WAL Segment Missing (Replica Sync হারিয়েছে)

```bash
# Replica log এ:
# ERROR: requested WAL segment 000000010000000000000050 has already been removed

# Primary তে WAL segment নেই কারণ:
# ① wal_keep_size কম ছিল
# ② Replication slot নেই, lag বেশি হয়ে গেছে

# Fix: Re-clone করতে হবে
sudo systemctl stop postgresql-16
sudo rm -rf /var/lib/pgsql/16/data/*
sudo -u postgres pg_basebackup \
    -h 172.16.93.140 -U replicator \
    -D /var/lib/pgsql/16/data \
    -Xs -R -P
sudo systemctl start postgresql-16

# Prevention:
# Primary তে wal_keep_size = 2GB বা replication slot ব্যবহার করো
```

### Scenario 5: Split Brain Prevention

```
Split Brain: দুটো node মনে করছে দুজনেই Primary
→ দুটো node তে data write হচ্ছে → data diverge → corruption

Patroni কীভাবে prevent করে:
  etcd majority (quorum) এর মাধ্যমে
  Leader lease: etcd তে TTL-based lock
  Network partition হলে: minority side লিখতে পারে না (etcd lock নেই)
  
  3-node etcd cluster:
    DC1: etcd1, PG+Patroni Node1 (leader)
    DC2: etcd2, PG+Patroni Node2
    DC3: etcd3 (tiebreaker)
  
  DC1 isolated:
    Node1 etcd quorum পাচ্ছে না (1/3)
    → Node1 leadership দেয় (demotes itself)
    DC2+DC3: quorum আছে (2/3) → নতুন leader elect
    → Split brain নেই
```

---

# 11. Percona for PostgreSQL

## 11.1 Percona Distribution for PostgreSQL — Overview

Percona Distribution for PostgreSQL হলো vanilla PostgreSQL এর একটা production-ready bundle। **Core PostgreSQL binary তে কোনো পরিবর্তন নেই** — শুধু curated extensions এবং tools যোগ করা।

```
Percona Distribution এ কী থাকে:

  ┌─────────────────────────────────────────────────────┐
  │             Vanilla PostgreSQL (unmodified)         │
  │         Core binary identical to PGDG              │
  └─────────────────────────────────────────────────────┘
            │ যোগ করা হয়েছে:
  ┌─────────▼───────────────────────────────────────────┐
  │  Extensions (curated):                              │
  │    pg_stat_monitor   ← Enhanced query analytics     │
  │    pgaudit           ← Audit logging                │
  │    pg_repack         ← Online table reorganization  │
  │    wal2json          ← WAL to JSON (CDC use cases)  │
  │    pg_trgm           ← Trigram text search          │
  │    pgBackRest        ← Backup & recovery            │
  │                                                     │
  │  HA Tools:                                          │
  │    Patroni           ← HA automation                │
  │    HAProxy           ← Load balancing               │
  │    etcd              ← Distributed config store     │
  │                                                     │
  │  Monitoring:                                        │
  │    PMM Client        ← Percona Monitoring           │
  └─────────────────────────────────────────────────────┘
```

### Install (Rocky Linux 9)

```bash
# Percona Repository
sudo dnf install -y https://repo.percona.com/yum/percona-release-latest.noarch.rpm
sudo percona-release setup ppg-16

# Install
sudo dnf install -y \
    percona-postgresql16-server \
    percona-postgresql16-contrib \
    percona-pg_stat_monitor16 \
    percona-pgbackrest \
    percona-patroni

# Initialize এবং start
sudo /usr/pgsql-16/bin/postgresql-16-setup initdb
sudo systemctl start postgresql-16
sudo systemctl enable postgresql-16
```

---

## 11.2 pg_stat_monitor — Deep Dive

Percona এর তৈরি, `pg_stat_statements` এর advanced replacement।

### pg_stat_statements এর কী কী limitation আছে

```
pg_stat_statements:
  ✅ Basic: calls, total_time, mean_time, rows
  ✅ Cache hit/miss
  ❌ Time-based history নেই
     → "কোন সময়ে slow হয়েছিল?" জানা যায় না
  ❌ Execution plan capture নেই
     → Slow query দেখলে manually EXPLAIN চালাতে হয়
  ❌ Response time distribution নেই
     → P95/P99 latency জানা যায় না
  ❌ Per-client-IP breakdown নেই
     → কোন app server থেকে slow query আসছে জানা যায় না
  ❌ Error tracking নেই

pg_stat_monitor (Percona):
  ✅ সব উপরের + নিচের সব
  ✅ Time buckets → "কোন সময়ে slow ছিল" history
  ✅ Execution plan auto-capture
  ✅ P50/P95/P99 response time percentiles
  ✅ Histogram (response time distribution)
  ✅ Per-client-IP breakdown
  ✅ Per-user breakdown
  ✅ Error count per query
  ✅ PMM QAN এর native data source
```

### Configuration

```bash
# Install
sudo dnf install -y percona-pg_stat_monitor16
```

```ini
# postgresql.conf
shared_preload_libraries = 'pg_stat_monitor'
# pg_stat_statements replace করো — এই দুটো একসাথে না

pg_stat_monitor.pgsm_bucket_time         = 60
# Time bucket size in seconds
# 60 seconds = প্রতি মিনিটে একটা bucket
# কম হলে → বেশি granular কিন্তু বেশি memory

pg_stat_monitor.pgsm_max_buckets         = 10
# কতটা bucket রাখবে
# 10 buckets × 60s = 10 মিনিটের history

pg_stat_monitor.pgsm_enable_query_plan   = yes
# Execution plan capture করো
# Memory খরচ বাড়ে কিন্তু debugging invaluable

pg_stat_monitor.pgsm_query_max_len       = 2048
# Maximum query text length

pg_stat_monitor.pgsm_histogram_min       = 1
pg_stat_monitor.pgsm_histogram_max       = 100000
pg_stat_monitor.pgsm_histogram_buckets   = 20
# Response time histogram: 1ms থেকে 100s, 20টা bucket

pg_stat_monitor.pgsm_track               = all
# top: top-level queries শুধু
# all: nested queries ও (functions, procedures এর ভেতরে)

pg_stat_monitor.pgsm_normalized_query    = yes
# Parameters normalize করে: WHERE id = $1 (না: WHERE id = 5)
# Same query different params → একটাই entry
```

```bash
sudo systemctl restart postgresql-16
```

```sql
CREATE EXTENSION pg_stat_monitor;
```

### Key Queries

```sql
-- Slowest queries (current bucket)
SELECT
    bucket_start_time                               AS time_window,
    LEFT(query, 100)                                AS query,
    calls,
    ROUND(mean_exec_time::numeric, 2)               AS avg_ms,
    ROUND(min_exec_time::numeric, 2)                AS min_ms,
    ROUND(max_exec_time::numeric, 2)                AS max_ms,
    ROUND(p99_exec_time::numeric, 2)                AS p99_ms,
    rows,
    datname                                         AS database,
    username,
    client_ip
FROM pg_stat_monitor
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Time-based analysis: "কোন সময়ে slow হয়েছিল?"
SELECT
    bucket_start_time,
    calls,
    ROUND(mean_exec_time::numeric, 2)  AS avg_ms,
    ROUND(max_exec_time::numeric, 2)   AS max_ms
FROM pg_stat_monitor
WHERE query ILIKE '%SELECT%orders%'
ORDER BY bucket_start_time DESC;

-- P50/P95/P99 percentiles
SELECT
    LEFT(query, 80)                                 AS query,
    calls,
    ROUND(p50_exec_time::numeric, 2)                AS p50_ms,
    ROUND(p95_exec_time::numeric, 2)                AS p95_ms,
    ROUND(p99_exec_time::numeric, 2)                AS p99_ms
FROM pg_stat_monitor
ORDER BY p99_exec_time DESC
LIMIT 10;

-- Execution plan দেখো (plan capture চালু থাকলে)
SELECT
    LEFT(query, 80)                                 AS query,
    calls,
    ROUND(mean_exec_time::numeric, 2)               AS avg_ms,
    query_plan
FROM pg_stat_monitor
WHERE query_plan IS NOT NULL
  AND mean_exec_time > 100
ORDER BY mean_exec_time DESC
LIMIT 5;

-- Error-prone queries
SELECT
    LEFT(query, 80)                                 AS query,
    calls,
    errors,
    ROUND(errors * 100.0 / NULLIF(calls, 0), 2)    AS error_pct
FROM pg_stat_monitor
WHERE errors > 0
ORDER BY error_pct DESC;

-- Per client IP: কোন app server থেকে slow query
SELECT
    client_ip,
    datname,
    username,
    count(DISTINCT query)                           AS unique_queries,
    sum(calls)                                      AS total_calls,
    ROUND(avg(mean_exec_time)::numeric, 2)          AS avg_ms
FROM pg_stat_monitor
GROUP BY client_ip, datname, username
ORDER BY total_calls DESC;

-- Cache efficiency per query
SELECT
    LEFT(query, 60)                                 AS query,
    calls,
    ROUND(
        shared_blks_hit * 100.0 /
        NULLIF(shared_blks_hit + shared_blks_read, 0),
    2)                                              AS cache_hit_pct
FROM pg_stat_monitor
WHERE shared_blks_hit + shared_blks_read > 0
ORDER BY cache_hit_pct ASC
LIMIT 10;  -- সবচেয়ে কম cache hit → index বা shared_buffers সমস্যা

-- Reset
SELECT pg_stat_monitor_reset();
```

---

## 11.3 Percona Monitoring and Management (PMM)

PMM হলো Percona এর open-source monitoring platform। MySQL এবং PostgreSQL দুটোই monitor করে।

```
PMM = Grafana + Prometheus + VictoriaMetrics + QAN (Query Analytics)

কী monitor করে:
  PostgreSQL:
    ✅ Connection, TPS, QPS
    ✅ Shared buffers hit rate
    ✅ Replication lag (bytes + time)
    ✅ VACUUM/Autovacuum activity
    ✅ Lock waits
    ✅ Table/Index bloat
    ✅ XID wraparound risk
    ✅ Query analytics (QAN) — pg_stat_monitor থেকে

  OS Level (সব server):
    ✅ CPU, Memory, Disk I/O, Network
    ✅ Load average
    ✅ Filesystem usage

  MySQL/MariaDB: ✅ (same platform, MySQL ও monitor করা যায়)
```

### Architecture

```
  ┌───────────────────────────────────────────────────┐
  │                  PMM Server                       │
  │  Grafana UI (HTTPS 443)                           │
  │  Prometheus + VictoriaMetrics (metrics storage)   │
  │  Query Analytics (QAN)                            │
  │  Alertmanager (notifications)                     │
  └──────────────────────┬────────────────────────────┘
                         │ HTTPS metrics push
       ┌─────────────────┼─────────────────┐
       ▼                 ▼                 ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │PMM Client│    │PMM Client│    │PMM Client│
  │PG Primary│    │PG Standby│    │MySQL     │
  │          │    │          │    │(optional)│
  └──────────┘    └──────────┘    └──────────┘
```

---

## 11.4 PMM Setup — End-to-End

### PMM Server Install (Docker)

```bash
# Docker install
sudo dnf install -y docker-ce
sudo systemctl start docker && sudo systemctl enable docker

# PMM Server container
docker run -d \
    --name pmm-server \
    --restart always \
    -p 80:80 \
    -p 443:443 \
    -v pmm-data:/srv \
    percona/pmm-server:2

# ~60 second অপেক্ষা করো
# Browser এ: https://pmm-server-ip
# Login: admin / admin → password change করো

# Health check
curl -k https://localhost/v1/readyz
```

### PMM Client (প্রতিটা PostgreSQL Server এ)

```bash
# Install
sudo dnf install -y https://repo.percona.com/yum/percona-release-latest.noarch.rpm
sudo percona-release enable pmm2-client
sudo dnf install -y pmm2-client

# PMM Server এ register করো
sudo pmm-admin config \
    --server-insecure-tls \
    --server-url=https://admin:YourPMMPassword@172.16.93.150

# PostgreSQL service add করো
sudo pmm-admin add postgresql \
    --username=pmm_monitor \
    --password=PMM@Monitor2024! \
    --service-name=pg-primary-140 \
    --host=127.0.0.1 \
    --port=5432 \
    --query-source=pgstatmonitor    # ← pg_stat_monitor ব্যবহার করো

# Status
sudo pmm-admin list
# pg-primary-140    PostgreSQL    OK
```

```sql
-- PostgreSQL এ monitoring user তৈরি করো
CREATE USER pmm_monitor PASSWORD 'PMM@Monitor2024!';
GRANT pg_monitor TO pmm_monitor;
GRANT SELECT ON pg_stat_monitor TO pmm_monitor;
-- pg_hba.conf এ:
-- host all pmm_monitor 127.0.0.1/32 scram-sha-256
```

### PMM Dashboard Overview

```
PostgreSQL Overview:
  → Active connections (trend)
  → Transactions per second (TPS)
  → Queries per second
  → Shared buffers hit rate (99%+ alert)
  → Temp file usage (work_mem টিউনিং indicator)

PostgreSQL Instance Summary:
  → Database size growth chart
  → Table bloat by table
  → Dead tuples per table (autovacuum indicator)
  → Autovacuum activity timeline

PostgreSQL Replication:
  → Streaming status (streaming/catchup/down)
  → Lag bytes chart
  → Replication slot WAL retention

Node Summary (OS):
  → CPU usage, iowait
  → Memory: used/cached/free
  → Disk I/O: read/write MB/s, IOPS
  → Network: bytes in/out

Query Analytics (QAN):
  → Top queries by total time, calls, avg time
  → Per-query execution plan (from pg_stat_monitor)
  → Response time histogram (P50/P75/P95/P99)
  → Filter by: user, database, client IP, time range
  → Historical comparison: "আগে এই query কতক্ষণ লাগত?"
```

### PMM Alerting Configure করো

```
PMM UI → Alerting → Alert Rules:

Built-in Templates:
  PostgreSQL:
    ← PostgreSQL replication lag > 60s
    ← Connection pool > 80% utilized
    ← Shared buffer hit rate < 95%
    ← XID age > 1.5 billion (wraparound risk!)
    ← Autovacuum not running for > 1 hour
    ← Table bloat > 50% of table size
    ← Long-running query > 10 minutes

  OS:
    ← Disk usage > 85%
    ← CPU > 90% for > 5 minutes
    ← Memory available < 10%

Alert Channels:
  Email, Slack, PagerDuty, OpsGenie
  → Alerting → Notification Channels এ configure করো
```

---

## 11.5 PMM Query Analytics (QAN) — ব্যবহার করার Guide

```
PMM UI → Query Analytics:

Step 1: Time range select করো (Last 1h, 6h, 24h, custom)
Step 2: Service select করো (pg-primary-140)
Step 3: Query list দেখো (default: total load দিয়ে sort)

Sort করো:
  Total Load    → server এ সবচেয়ে বেশি impact
  Avg Time      → প্রতিটা execution এ সবচেয়ে slow
  Call Count    → সবচেয়ে বেশিবার চলে

Per-query details:
  Overview tab:
    → Calls, avg time, total time, P95/P99
    → Time series chart (কখন spike হয়েছিল)
    → Breakdown by user / client IP

  Explain tab:
    → Execution plan (pg_stat_monitor থেকে auto-captured)
    → Plan tree visual representation
    → "Seq Scan on large table" দেখলে → index দাও

  History tab:
    → "এই query কি আগেও slow ছিল?"
    → Bucket-by-bucket comparison

Workflow:
  1. High load query identify করো
  2. Explain tab এ Seq Scan বা expensive operation দেখো
  3. EXPLAIN (ANALYZE, BUFFERS) manually চালিয়ে confirm করো
  4. Index দাও / query rewrite করো
  5. PMM তে improvement monitor করো
```

---

## 11.6 pg_stat_monitor vs pg_stat_statements

| Feature | pg_stat_statements | pg_stat_monitor |
|---|---|---|
| Installation | Built-in, PGDG | Percona / PGDG package |
| Memory overhead | Low | Medium |
| Basic metrics (calls, time, rows) | ✅ | ✅ |
| WAL metrics | ✅ (PG 13+) | ✅ |
| JIT statistics | ✅ | ✅ |
| Time bucket history | ❌ | ✅ |
| "কখন slow ছিল" | ❌ | ✅ |
| Execution plan capture | ❌ | ✅ |
| P50/P95/P99 percentiles | ❌ | ✅ |
| Response time histogram | ❌ | ✅ |
| Per-client-IP breakdown | ❌ | ✅ |
| Per-database breakdown | Limited | ✅ |
| Error tracking per query | ❌ | ✅ |
| PMM QAN native support | Limited | ✅ Native |
| Community support | PostgreSQL core | Percona + community |

**কোনটা ব্যবহার করবো:**
```
pg_stat_statements:
  → Minimal setup চাই
  → PMM ব্যবহার করছো না
  → Minimum overhead priority
  → Basic slow query tracking যথেষ্ট

pg_stat_monitor:
  → PMM ব্যবহার করছো (native integration, full QAN)
  → "কোন সময়ে slow হয়েছিল" জানতে চাও
  → Execution plan auto-capture দরকার
  → P95/P99 latency SLA track করতে চাও
  → কোন client IP বা user slow query করছে জানতে চাও
  → Production environment, detailed analytics দরকার
```

---

## 11.7 Percona Toolkit for PostgreSQL

```bash
sudo dnf install -y percona-toolkit
```

### pg_repack — Online Table Reorganization

```bash
sudo dnf install -y pg_repack_16

# VACUUM FULL এর alternative — Table lock নেয় না!
# Table bloated হলে:
pg_repack -h localhost -U postgres -d mydb -t users
# নতুন table তৈরি করে, data copy করে, atomically swap করে
# Production traffic চলতে থাকে

# Full database repack
pg_repack -h localhost -U postgres -d mydb

# Index only repack (REINDEX CONCURRENTLY এর alternative)
pg_repack -h localhost -U postgres -d mydb --only-indexes -t users
```

**VACUUM FULL vs pg_repack:**
```
VACUUM FULL:
  ✅ Built-in, simple
  ❌ ACCESS EXCLUSIVE lock → পুরো table block
  ❌ Production downtime

pg_repack:
  ✅ Online, no downtime
  ✅ Concurrent reads/writes continue
  ❌ Requires extension install
  ❌ Extra disk space needed (temporary copy)
  → Production এ সবসময় pg_repack ব্যবহার করো
```

### Other Useful Tools

```bash
# CREATE INDEX CONCURRENTLY — lock ছাড়া index তৈরি
CREATE INDEX CONCURRENTLY idx_email ON users(email);
-- Regular CREATE INDEX: ShareLock নেয়, writes block হয়
-- CONCURRENTLY: Table live থাকে, slower কিন্তু safe

# DROP INDEX CONCURRENTLY
DROP INDEX CONCURRENTLY idx_old_index;

# REINDEX CONCURRENTLY (PG 12+)
REINDEX INDEX CONCURRENTLY idx_email;
REINDEX TABLE CONCURRENTLY users;
```

---

## 11.8 Percona vs Vanilla PostgreSQL

```
Vanilla PostgreSQL (PGDG):
  ✅ Official, community supported
  ✅ Latest features সবার আগে পাওয়া যায়
  ✅ Minimal footprint
  ✅ Simple deployment
  ✅ Standard package managers (dnf, apt)
  → ছোট থেকে medium deployment
  → Community support যথেষ্ট হলে
  → Extra tools manually install করতে পারো

Percona Distribution for PostgreSQL:
  ✅ Vanilla PG + curated toolset একসাথে
  ✅ pg_stat_monitor, pgBackRest, Patroni একসাথে install
  ✅ PMM দিয়ে professional monitoring
  ✅ Percona support subscription available
  ✅ Enterprise-grade production setup
  ✅ Tested combination (extension compatibility guaranteed)
  → Production environment
  → PMM monitoring চাইলে
  → Percona support contract প্রয়োজন হলে
  → Operations team এর জন্য turnkey solution

Important:
  Percona Distribution = Vanilla PG + Tools
  PostgreSQL binary পরিবর্তন হয় না
  Backup compatibility: same as vanilla
  যেকোনো সময় vanilla এ switch করা যায়
  Upgrade path: identical to vanilla
```

**Production Setup Recommendation:**
```
Small team, simple app:
  PGDG PostgreSQL 16
  + pgBackRest (backup)
  + Patroni (HA) — if needed
  + pgBouncer (connection pooling)

Medium/Large team, Production:
  Percona Distribution for PostgreSQL 16
  + PMM (monitoring + alerting)
  + pg_stat_monitor (query analytics)
  + pgBackRest (automated backup, PITR)
  + Patroni + etcd + HAProxy (HA cluster)
  + pgBouncer (connection pooling)
  + pg_repack (online table maintenance)
```

---

*PostgreSQL 16 | Rocky Linux 9 | Complete Study Reference*
*Percona Distribution | pgBackRest | Barman | Patroni | pgBouncer | pgPool-II | PMM*

---

# 12. PL/pgSQL — Functions, Procedures & Triggers

## 12.1 PL/pgSQL কী এবং কেন

PL/pgSQL হলো PostgreSQL এর built-in procedural language। SQL এর মধ্যে variables, loops, conditions, exception handling সব করা যায়।

```
কখন PL/pgSQL দরকার:
  ① Business logic database level এ রাখতে চাইলে
     (application সব ভাষায় same logic কাজ করবে)
  ② Multiple SQL statements এক unit এ চালাতে চাইলে
  ③ Complex validation, transformation
  ④ Triggers — data change এ automatically কিছু করতে
  ⑤ Batch processing — loop করে data process

MySQL vs PostgreSQL:
  MySQL:      Stored Procedure এবং Function আলাদা concept
  PostgreSQL: Function দিয়ে দুটোই করা যায়
              PROCEDURE keyword PG 11+ এ আছে (transaction control সহ)
```

---

## 12.2 Function — Basic Structure

```sql
-- Basic structure
CREATE OR REPLACE FUNCTION function_name(
    param1 datatype,
    param2 datatype DEFAULT default_value
)
RETURNS return_type
LANGUAGE plpgsql
AS $$
DECLARE
    -- variable declarations
    variable_name datatype;
    variable_name datatype := initial_value;
BEGIN
    -- logic here
    RETURN value;
END;
$$;
```

### Hands-on: প্রথম Function তৈরি করো

```sql
-- ─── Example 1: Simple function ───
CREATE OR REPLACE FUNCTION add_numbers(a INTEGER, b INTEGER)
RETURNS INTEGER
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN a + b;
END;
$$;

-- Call করো
SELECT add_numbers(10, 20);         -- 30
SELECT add_numbers(a => 5, b => 3); -- named parameters

-- ─── Example 2: Variable ব্যবহার ───
CREATE OR REPLACE FUNCTION get_full_name(user_id INTEGER)
RETURNS TEXT
LANGUAGE plpgsql
AS $$
DECLARE
    v_first  TEXT;
    v_last   TEXT;
    v_full   TEXT;
BEGIN
    SELECT first_name, last_name
    INTO v_first, v_last
    FROM users
    WHERE id = user_id;

    -- NULL check
    IF v_first IS NULL THEN
        RETURN 'User not found';
    END IF;

    v_full := v_first || ' ' || v_last;
    RETURN v_full;
END;
$$;

SELECT get_full_name(5);

-- ─── Example 3: SQL Function (simpler, plpgsql নয়) ───
-- Simple SQL দিয়েই হলে LANGUAGE sql ব্যবহার করো (faster)
CREATE OR REPLACE FUNCTION get_user_order_count(p_user_id INTEGER)
RETURNS BIGINT
LANGUAGE sql
STABLE    -- same input = same output, no side effects
AS $$
    SELECT COUNT(*) FROM orders WHERE user_id = p_user_id;
$$;

SELECT get_user_order_count(5);
```

---

## 12.3 Control Structures — IF, LOOP, CASE

```sql
-- ─── IF / ELSIF / ELSE ───
CREATE OR REPLACE FUNCTION classify_order(p_total NUMERIC)
RETURNS TEXT
LANGUAGE plpgsql
AS $$
BEGIN
    IF p_total >= 10000 THEN
        RETURN 'platinum';
    ELSIF p_total >= 5000 THEN
        RETURN 'gold';
    ELSIF p_total >= 1000 THEN
        RETURN 'silver';
    ELSE
        RETURN 'bronze';
    END IF;
END;
$$;

SELECT classify_order(7500);  -- gold

-- ─── CASE ───
CREATE OR REPLACE FUNCTION day_type(p_date DATE)
RETURNS TEXT
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN CASE EXTRACT(DOW FROM p_date)
        WHEN 0 THEN 'Sunday'
        WHEN 6 THEN 'Saturday'
        ELSE 'Weekday'
    END;
END;
$$;

-- ─── LOOP ───
CREATE OR REPLACE FUNCTION generate_series_sum(p_n INTEGER)
RETURNS INTEGER
LANGUAGE plpgsql
AS $$
DECLARE
    v_sum   INTEGER := 0;
    v_i     INTEGER := 1;
BEGIN
    LOOP
        EXIT WHEN v_i > p_n;    -- exit condition
        v_sum := v_sum + v_i;
        v_i   := v_i + 1;
    END LOOP;
    RETURN v_sum;
END;
$$;

SELECT generate_series_sum(100);  -- 5050

-- ─── FOR LOOP ───
CREATE OR REPLACE FUNCTION factorial(p_n INTEGER)
RETURNS BIGINT
LANGUAGE plpgsql
AS $$
DECLARE
    v_result BIGINT := 1;
BEGIN
    FOR i IN 1..p_n LOOP
        v_result := v_result * i;
    END LOOP;
    RETURN v_result;
END;
$$;

SELECT factorial(10);  -- 3628800

-- ─── FOR LOOP over query results ───
CREATE OR REPLACE FUNCTION update_user_tiers()
RETURNS INTEGER
LANGUAGE plpgsql
AS $$
DECLARE
    v_rec    RECORD;
    v_count  INTEGER := 0;
    v_tier   TEXT;
BEGIN
    FOR v_rec IN
        SELECT u.id, COALESCE(SUM(o.total), 0) AS total_spent
        FROM users u
        LEFT JOIN orders o ON u.id = o.user_id
        GROUP BY u.id
    LOOP
        v_tier := CASE
            WHEN v_rec.total_spent >= 10000 THEN 'platinum'
            WHEN v_rec.total_spent >= 5000  THEN 'gold'
            WHEN v_rec.total_spent >= 1000  THEN 'silver'
            ELSE 'bronze'
        END;

        UPDATE users SET tier = v_tier WHERE id = v_rec.id;
        v_count := v_count + 1;
    END LOOP;

    RETURN v_count;  -- কতটা row update হয়েছে
END;
$$;

SELECT update_user_tiers();

-- ─── WHILE LOOP ───
CREATE OR REPLACE FUNCTION wait_for_condition()
RETURNS VOID
LANGUAGE plpgsql
AS $$
DECLARE
    v_retries INTEGER := 0;
BEGIN
    WHILE v_retries < 5 LOOP
        -- কিছু check করো
        v_retries := v_retries + 1;
        PERFORM pg_sleep(1);
    END LOOP;
END;
$$;
```

---

## 12.4 Exception Handling

```sql
-- ─── Basic Exception Handling ───
CREATE OR REPLACE FUNCTION safe_divide(a NUMERIC, b NUMERIC)
RETURNS NUMERIC
LANGUAGE plpgsql
AS $$
BEGIN
    IF b = 0 THEN
        RAISE EXCEPTION 'Division by zero: b cannot be 0';
    END IF;
    RETURN a / b;

EXCEPTION
    WHEN division_by_zero THEN
        RAISE WARNING 'Caught division by zero';
        RETURN NULL;
    WHEN OTHERS THEN
        RAISE EXCEPTION 'Unexpected error: %', SQLERRM;
END;
$$;

SELECT safe_divide(10, 2);   -- 5
SELECT safe_divide(10, 0);   -- NULL (caught)

-- ─── RAISE levels ───
-- RAISE DEBUG   → log_min_messages = debug এ দেখা যায়
-- RAISE LOG     → server log এ
-- RAISE INFO    → client এ দেখায়
-- RAISE NOTICE  → client এ দেখায় (common debug tool)
-- RAISE WARNING → client এ warning
-- RAISE EXCEPTION → error, transaction rollback

CREATE OR REPLACE FUNCTION process_payment(
    p_user_id INTEGER,
    p_amount  NUMERIC
)
RETURNS TEXT
LANGUAGE plpgsql
AS $$
DECLARE
    v_balance NUMERIC;
BEGIN
    -- Balance check
    SELECT balance INTO v_balance FROM accounts WHERE user_id = p_user_id;

    IF NOT FOUND THEN
        RAISE EXCEPTION 'User % not found', p_user_id
            USING ERRCODE = 'P0001';   -- custom error code
    END IF;

    IF v_balance < p_amount THEN
        RAISE EXCEPTION 'Insufficient balance: has %, needs %',
            v_balance, p_amount
            USING ERRCODE = 'P0002',
                  HINT = 'Please top up your account';
    END IF;

    -- Deduct balance
    UPDATE accounts SET balance = balance - p_amount
    WHERE user_id = p_user_id;

    RAISE NOTICE 'Payment of % processed for user %', p_amount, p_user_id;
    RETURN 'SUCCESS';

EXCEPTION
    WHEN SQLSTATE 'P0001' THEN
        RETURN 'ERROR: User not found';
    WHEN SQLSTATE 'P0002' THEN
        RETURN 'ERROR: Insufficient balance';
    WHEN unique_violation THEN
        RETURN 'ERROR: Duplicate transaction';
    WHEN OTHERS THEN
        -- Error details
        RAISE WARNING 'Error: %, Detail: %, Hint: %',
            SQLERRM, SQLSTATE, PG_EXCEPTION_HINT;
        RETURN 'ERROR: ' || SQLERRM;
END;
$$;

SELECT process_payment(5, 100.00);
```

---

## 12.5 Stored Procedures (PostgreSQL 11+)

Function এর সাথে পার্থক্য: Procedure এ `COMMIT` এবং `ROLLBACK` করা যায় (transaction control)।

```sql
-- ─── Basic Procedure ───
CREATE OR REPLACE PROCEDURE transfer_money(
    p_from_id INTEGER,
    p_to_id   INTEGER,
    p_amount  NUMERIC
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_from_balance NUMERIC;
BEGIN
    -- Lock করো (deadlock এড়াতে smaller ID first)
    IF p_from_id < p_to_id THEN
        SELECT balance INTO v_from_balance
        FROM accounts WHERE user_id = p_from_id FOR UPDATE;

        PERFORM balance FROM accounts WHERE user_id = p_to_id FOR UPDATE;
    ELSE
        PERFORM balance FROM accounts WHERE user_id = p_to_id FOR UPDATE;

        SELECT balance INTO v_from_balance
        FROM accounts WHERE user_id = p_from_id FOR UPDATE;
    END IF;

    IF v_from_balance < p_amount THEN
        RAISE EXCEPTION 'Insufficient funds: balance=%, amount=%',
            v_from_balance, p_amount;
    END IF;

    UPDATE accounts SET balance = balance - p_amount WHERE user_id = p_from_id;
    UPDATE accounts SET balance = balance + p_amount WHERE user_id = p_to_id;

    INSERT INTO transaction_log (from_id, to_id, amount, created_at)
    VALUES (p_from_id, p_to_id, p_amount, NOW());

    COMMIT;  -- Procedure এ COMMIT করা যায়!
    RAISE NOTICE 'Transfer of % from user % to user % completed',
        p_amount, p_from_id, p_to_id;

EXCEPTION
    WHEN OTHERS THEN
        ROLLBACK;  -- Error এ rollback
        RAISE;
END;
$$;

-- Procedure call করো (SELECT নয়, CALL দিয়ে)
CALL transfer_money(1, 2, 500.00);

-- Function vs Procedure:
-- Function:   SELECT দিয়ে call, value return করে, transaction control নেই
-- Procedure:  CALL দিয়ে call, COMMIT/ROLLBACK করতে পারে
```

---

## 12.6 Return Types — TABLE, SETOF, RECORD

```sql
-- ─── RETURNS TABLE ───
CREATE OR REPLACE FUNCTION get_user_orders(p_user_id INTEGER)
RETURNS TABLE(
    order_id    INTEGER,
    total       NUMERIC,
    status      TEXT,
    created_at  TIMESTAMPTZ
)
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY
        SELECT o.id, o.total, o.status, o.created_at
        FROM orders o
        WHERE o.user_id = p_user_id
        ORDER BY o.created_at DESC;
END;
$$;

-- Table এর মতো query করো
SELECT * FROM get_user_orders(5);
SELECT order_id, total FROM get_user_orders(5) WHERE status = 'pending';

-- ─── RETURNS SETOF ───
CREATE OR REPLACE FUNCTION get_active_users()
RETURNS SETOF users    -- পুরো users table row return করে
LANGUAGE sql
AS $$
    SELECT * FROM users WHERE status = 'active';
$$;

SELECT id, name FROM get_active_users();

-- ─── OUT parameters ───
CREATE OR REPLACE FUNCTION get_user_stats(
    p_user_id   INTEGER,
    OUT order_count BIGINT,
    OUT total_spent NUMERIC,
    OUT avg_order   NUMERIC
)
LANGUAGE plpgsql
AS $$
BEGIN
    SELECT COUNT(*), SUM(total), AVG(total)
    INTO order_count, total_spent, avg_order
    FROM orders
    WHERE user_id = p_user_id;
END;
$$;

SELECT * FROM get_user_stats(5);
-- order_count | total_spent | avg_order
```

---

## 12.7 Triggers — Data Change এ Automatic Action

```sql
-- ─── Trigger এর structure ───
-- Step 1: Trigger Function তৈরি করো (RETURNS TRIGGER)
-- Step 2: Trigger তৈরি করো (কোন table, কখন, কোন event)

-- ─── Example 1: updated_at auto-update ───
-- (MySQL এর ON UPDATE CURRENT_TIMESTAMP এর equivalent)
CREATE OR REPLACE FUNCTION fn_set_updated_at()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    NEW.updated_at := NOW();
    RETURN NEW;    -- NEW = নতুন row (UPDATE/INSERT এ)
END;
$$;

CREATE TRIGGER trg_users_updated_at
    BEFORE UPDATE ON users           -- UPDATE এর আগে
    FOR EACH ROW                     -- প্রতিটা row এ
    EXECUTE FUNCTION fn_set_updated_at();

-- Test করো
UPDATE users SET name = 'Alice New' WHERE id = 5;
SELECT updated_at FROM users WHERE id = 5;
-- updated_at automatically NOW() হয়ে যাবে ✅

-- ─── Example 2: Audit Log Trigger ───
CREATE TABLE audit_log (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    table_name  TEXT,
    operation   TEXT,      -- INSERT / UPDATE / DELETE
    old_data    JSONB,     -- পুরনো data
    new_data    JSONB,     -- নতুন data
    changed_by  TEXT,      -- কোন user
    changed_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE OR REPLACE FUNCTION fn_audit_log()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO audit_log (table_name, operation, new_data, changed_by)
        VALUES (TG_TABLE_NAME, 'INSERT', to_jsonb(NEW), current_user);
        RETURN NEW;

    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO audit_log (table_name, operation, old_data, new_data, changed_by)
        VALUES (TG_TABLE_NAME, 'UPDATE', to_jsonb(OLD), to_jsonb(NEW), current_user);
        RETURN NEW;

    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO audit_log (table_name, operation, old_data, changed_by)
        VALUES (TG_TABLE_NAME, 'DELETE', to_jsonb(OLD), current_user);
        RETURN OLD;
    END IF;
END;
$$;

-- orders table এ audit করো
CREATE TRIGGER trg_orders_audit
    AFTER INSERT OR UPDATE OR DELETE ON orders
    FOR EACH ROW
    EXECUTE FUNCTION fn_audit_log();

-- Test করো
INSERT INTO orders (user_id, total, status) VALUES (1, 500, 'pending');
UPDATE orders SET status = 'confirmed' WHERE id = 1;
DELETE FROM orders WHERE id = 1;

SELECT * FROM audit_log ORDER BY changed_at DESC LIMIT 5;
-- সব changes recorded আছে ✅

-- ─── Example 3: Validation Trigger ───
CREATE OR REPLACE FUNCTION fn_validate_order()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    -- Total negative হতে পারবে না
    IF NEW.total < 0 THEN
        RAISE EXCEPTION 'Order total cannot be negative: %', NEW.total;
    END IF;

    -- Status transition validation
    IF TG_OP = 'UPDATE' THEN
        IF OLD.status = 'completed' AND NEW.status != 'completed' THEN
            RAISE EXCEPTION 'Cannot change status of completed order';
        END IF;
        IF OLD.status = 'cancelled' THEN
            RAISE EXCEPTION 'Cannot modify cancelled order';
        END IF;
    END IF;

    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_orders_validate
    BEFORE INSERT OR UPDATE ON orders
    FOR EACH ROW
    EXECUTE FUNCTION fn_validate_order();

-- Test invalid operations
UPDATE orders SET total = -100 WHERE id = 1;
-- ERROR: Order total cannot be negative ✅

-- ─── Trigger Variables ───
-- TG_OP:        'INSERT', 'UPDATE', 'DELETE', 'TRUNCATE'
-- TG_TABLE_NAME: table এর নাম
-- TG_WHEN:      'BEFORE', 'AFTER', 'INSTEAD OF'
-- TG_LEVEL:     'ROW', 'STATEMENT'
-- NEW:          নতুন row (INSERT/UPDATE এ)
-- OLD:          পুরনো row (UPDATE/DELETE এ)

-- ─── Trigger list দেখো ───
SELECT trigger_name, event_manipulation, event_object_table,
    action_timing, action_orientation
FROM information_schema.triggers
WHERE trigger_schema = 'public'
ORDER BY event_object_table, trigger_name;

-- ─── Trigger disable / enable করো ───
ALTER TABLE orders DISABLE TRIGGER trg_orders_audit;   -- একটা disable
ALTER TABLE orders ENABLE  TRIGGER trg_orders_audit;   -- enable
ALTER TABLE orders DISABLE TRIGGER ALL;                -- সব disable
ALTER TABLE orders ENABLE  TRIGGER ALL;
```

---

## 12.8 Function Performance এবং Volatility

```sql
-- Function এর volatility PostgreSQL এর optimization এ কাজ লাগে

-- VOLATILE (default):
--   প্রতিবার call এ আলাদা result হতে পারে
--   Side effects আছে (INSERT/UPDATE/DELETE করে)
--   Planner cache করে না
CREATE FUNCTION insert_log() RETURNS VOID VOLATILE ...

-- STABLE:
--   Same transaction এ same input = same output
--   Table data পড়তে পারে কিন্তু modify করে না
--   Index scan এ use হতে পারে
CREATE FUNCTION get_user(INTEGER) RETURNS users STABLE ...

-- IMMUTABLE:
--   Same input সবসময় same output (global state depend করে না)
--   Compile-time constant folding হয়
--   Index এ use করা যায়
CREATE FUNCTION lower_email(TEXT) RETURNS TEXT IMMUTABLE ...

-- Security
-- SECURITY DEFINER: function owner এর permission এ চলে (not caller)
-- SECURITY INVOKER: caller এর permission এ চলে (default)

-- Performance
-- PARALLEL SAFE:   Parallel query তে use করা যায়
-- PARALLEL UNSAFE: Parallel এ use করা যাবে না (default for PL/pgSQL)
-- PARALLEL RESTRICTED: Parallel leader process এ চলে

-- ─── Function list দেখো ───
SELECT routine_name, routine_type, data_type,
    external_language, security_type
FROM information_schema.routines
WHERE routine_schema = 'public'
ORDER BY routine_type, routine_name;

-- ─── Function drop করো ───
DROP FUNCTION IF EXISTS function_name(param_type);
DROP FUNCTION IF EXISTS add_numbers(INTEGER, INTEGER);

-- ─── Function এর definition দেখো ───
SELECT pg_get_functiondef(oid)
FROM pg_proc
WHERE proname = 'add_numbers';
-- অথবা psql এ: \sf add_numbers
```

---

## 12.9 Useful Built-in Functions — Quick Reference

```sql
-- ─── String Functions ───
SELECT LENGTH('hello');                           -- 5
SELECT UPPER('hello'), LOWER('WORLD');
SELECT TRIM('  hello  '), LTRIM, RTRIM;
SELECT SUBSTRING('hello world', 1, 5);            -- hello
SELECT REPLACE('hello world', 'world', 'PG');
SELECT REGEXP_REPLACE('abc123', '[0-9]', 'X', 'g'); -- abcXXX
SELECT SPLIT_PART('a,b,c', ',', 2);               -- b
SELECT STRING_AGG(name, ', ' ORDER BY name)       -- aggregate
       FROM users;
SELECT FORMAT('Hello %s, you have %s orders', name, count)
       FROM users_summary;

-- ─── Date/Time Functions ───
SELECT NOW(), CURRENT_TIMESTAMP, CURRENT_DATE;
SELECT EXTRACT(YEAR FROM NOW());
SELECT DATE_TRUNC('month', NOW());                -- মাসের শুরু
SELECT NOW() + INTERVAL '7 days';
SELECT AGE(NOW(), created_at) FROM users;         -- কতদিন হলো
SELECT TO_CHAR(NOW(), 'YYYY-MM-DD HH24:MI:SS');
SELECT TO_DATE('2024-03-08', 'YYYY-MM-DD');

-- ─── JSON Functions ───
SELECT jsonb_build_object('name', 'Alice', 'age', 30);
SELECT jsonb_agg(row_to_json(u)) FROM users u;    -- rows → JSON array
SELECT jsonb_array_elements('[1,2,3]'::jsonb);    -- array → rows
SELECT jsonb_object_keys('{"a":1,"b":2}'::jsonb); -- keys → rows

-- ─── Array Functions ───
SELECT ARRAY[1,2,3] || ARRAY[4,5];               -- concatenate
SELECT ARRAY_APPEND(ARRAY[1,2], 3);
SELECT ARRAY_REMOVE(ARRAY[1,2,3,2], 2);          -- [1,3]
SELECT ARRAY_LENGTH(ARRAY[1,2,3], 1);            -- 3
SELECT UNNEST(ARRAY[1,2,3]);                      -- array → rows
SELECT ARRAY_AGG(id ORDER BY id) FROM users;      -- rows → array

-- ─── Math Functions ───
SELECT ROUND(3.14159, 2);                         -- 3.14
SELECT CEIL(3.1), FLOOR(3.9);                     -- 4, 3
SELECT ABS(-5), SQRT(16), POWER(2, 10);
SELECT RANDOM();                                   -- 0.0 to 1.0
SELECT GENERATE_SERIES(1, 5);                     -- 1,2,3,4,5

-- ─── Type Conversion ───
SELECT '123'::INTEGER, '3.14'::NUMERIC;
SELECT CAST('2024-03-08' AS DATE);
SELECT TO_NUMBER('1,234.56', '9,999.99');
SELECT id::TEXT FROM users;
```


---

# 13. Major Version Upgrade

## 13.1 Upgrade Methods — Overview

PostgreSQL major version upgrade (e.g., PG 14 → PG 16) তিনটা উপায়ে করা যায়। প্রতিটার আলাদা tradeoff।

```
Method 1: pg_dump + pg_restore (Logical)
  কীভাবে: Old version dump → New version restore
  Downtime: Database size অনুযায়ী (1GB = ~5-10min)
  Risk: Low
  Cross-version: ✅ যেকোনো version থেকে যেকোনো version
  Use case: Small-medium DB, downtime window আছে

Method 2: pg_upgrade (In-place Physical Upgrade)
  কীভাবে: Data files directly upgrade করে
  Downtime: Minutes (data copy করে না, link করে)
  Risk: Medium (rollback harder)
  Cross-version: Only sequential versions safe (14→15→16)
  Use case: Large DB, minimal downtime চাই

Method 3: Logical Replication (Zero-downtime)
  কীভাবে: New version server এ replicate → cutover
  Downtime: Seconds (switchover time)
  Risk: Low (old server intact থাকে)
  Cross-version: ✅ PG 10+ থেকে
  Use case: Production, zero downtime required
```

---

## 13.2 Method 1 — pg_dump + pg_restore

সবচেয়ে সহজ এবং safe। Small database এর জন্য ideal।

```bash
# ─── Pre-upgrade checklist ───
# Old server (PG 14) এ:
psql -U postgres -c "SELECT version();"
psql -U postgres -c "\l"          -- database list
psql -U postgres -c "\du"         -- users list
psql -U postgres -c "SELECT COUNT(*) FROM pg_extension;" -- extensions

# ─── Step 1: Old server থেকে dump নাও ───
# Roles এবং tablespaces আগে dump করো
pg_dumpall -U postgres --globals-only > /backup/globals.sql

# প্রতিটা database dump করো
pg_dump -U postgres -Fd -j 4 -f /backup/mydb_dump mydb
# -Fd = directory format (parallel possible)
# -j 4 = 4 parallel workers

echo "Dump size: $(du -sh /backup/mydb_dump)"
echo "Dump completed: $(date)"

# ─── Step 2: New server এ PostgreSQL 16 install করো ───
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo dnf -qy module disable postgresql
sudo dnf install -y postgresql16-server postgresql16-contrib
sudo /usr/pgsql-16/bin/postgresql-16-setup initdb
sudo systemctl start postgresql-16

# ─── Step 3: Globals restore (roles, tablespaces) ───
psql -U postgres -f /backup/globals.sql
# Error দেখলে: role already exists → normal, ignore

# ─── Step 4: Database restore ───
createdb -U postgres mydb
time pg_restore -U postgres -d mydb -j 4 --verbose /backup/mydb_dump 2>&1 | \
    tee /tmp/restore.log

# Errors check করো
grep -i "error\|failed" /tmp/restore.log | head -20

# ─── Step 5: Verify ───
psql -U postgres -d mydb -c "SELECT COUNT(*) FROM orders;"
psql -U postgres -d mydb -c "SELECT COUNT(*) FROM users;"
# Old server এর count এর সাথে মেলাও!

# Extension verify করো
psql -U postgres -d mydb -c "\dx"
# Missing extension থাকলে:
psql -U postgres -d mydb -c "CREATE EXTENSION pgcrypto;"

# ─── Step 6: Statistics update করো ───
psql -U postgres -d mydb -c "ANALYZE VERBOSE;"

# ─── Step 7: Application connect করো, test করো ───
# Application → New server এ point করো
# Smoke test চালাও
```

---

## 13.3 Method 2 — pg_upgrade (Recommended for Large DB)

```bash
# ─── Pre-upgrade requirements ───
# ① Old এবং New PostgreSQL উভয়ই install থাকতে হবে (same server)
# ② Old server STOP থাকতে হবে
# ③ New cluster initialized কিন্তু empty

# ─── Step 1: উভয় version install করো ───
# PG 14 already আছে, PG 16 install করো:
sudo dnf install -y postgresql16-server postgresql16-contrib

# PG 16 initialize করো (start করো না)
sudo /usr/pgsql-16/bin/postgresql-16-setup initdb

# ─── Step 2: Old server stop করো ───
sudo systemctl stop postgresql-14

# Final checkpoint নাও
sudo -u postgres /usr/pgsql-14/bin/pg_ctl \
    -D /var/lib/pgsql/14/data \
    stop -m fast

# ─── Step 3: pg_upgrade compatibility check করো ───
sudo -u postgres /usr/pgsql-16/bin/pg_upgrade \
    --old-datadir=/var/lib/pgsql/14/data \
    --new-datadir=/var/lib/pgsql/16/data \
    --old-bindir=/usr/pgsql-14/bin \
    --new-bindir=/usr/pgsql-16/bin \
    --check    # ← শুধু check করে, upgrade করে না

# Output check করো:
# Clusters are compatible ✅
# অথবা error দেখাবে কী fix করতে হবে

# ─── Step 4: Actual upgrade চালাও ───
sudo -u postgres /usr/pgsql-16/bin/pg_upgrade \
    --old-datadir=/var/lib/pgsql/14/data \
    --new-datadir=/var/lib/pgsql/16/data \
    --old-bindir=/usr/pgsql-14/bin \
    --new-bindir=/usr/pgsql-16/bin \
    --jobs=4 \       # parallel (table statistics)
    --link           # ← Hard link (data copy করে না → FAST!)
                     # --link ছাড়া: data copy করে → slow কিন্তু safe
                     # --link দিলে: rollback সম্ভব না (old data shared)

# Progress দেখবে:
# Performing Consistency Checks...
# ok
# Upgrading Database Contents...
# Setting next transaction id and epoch for new cluster
# ...
# Upgrade Complete

# ─── Step 5: New server start করো ───
sudo systemctl start postgresql-16

# ─── Step 6: Statistics rebuild করো ───
# pg_upgrade statistics migrate করে না
sudo -u postgres /usr/pgsql-16/bin/vacuumdb \
    --all --analyze-in-stages -U postgres
# --analyze-in-stages: তিনটা stage এ করে, শুরুতে minimal stats → fast start

# ─── Step 7: Verify ───
psql -U postgres -c "SELECT version();"
psql -U postgres -d mydb -c "SELECT COUNT(*) FROM orders;"

# ─── Step 8: Old cluster cleanup ───
# Verify সব ঠিক আছে কয়েক দিন পরে:
sudo -u postgres ./delete_old_cluster.sh   # pg_upgrade তৈরি করে
# এটা old data directory delete করবে

# ─── Rollback (--link ছাড়া হলে) ───
sudo systemctl stop postgresql-16
sudo systemctl start postgresql-14
# Application → PG 14 তে point করো
# NOTE: --link দিয়ে upgrade করলে rollback সম্ভব না!
```

---

## 13.4 Method 3 — Logical Replication (Zero-Downtime)

Production এর জন্য সবচেয়ে safe। Old server চলতে থাকে, New server তে replicate, তারপর seconds এ cutover।

```bash
# ─── Environment ───
# Old: PG 14 — 172.16.93.140:5432 (production)
# New: PG 16 — 172.16.93.143:5432 (new server)

# ─── Step 1: New server এ PG 16 install এবং configure করো ───
# (New server 172.16.93.143 এ)
sudo dnf install -y postgresql16-server postgresql16-contrib
sudo /usr/pgsql-16/bin/postgresql-16-setup initdb

# New server এর postgresql.conf:
psql -U postgres -c "ALTER SYSTEM SET wal_level = 'logical';"
sudo systemctl restart postgresql-16

# ─── Step 2: Old server এ wal_level = logical set করো ───
# (Old server 172.16.93.140 এ — PG 14)
psql -U postgres -c "ALTER SYSTEM SET wal_level = 'logical';"
sudo systemctl restart postgresql-14
psql -U postgres -c "SHOW wal_level;"   -- logical confirm

# ─── Step 3: Schema copy করো (data ছাড়া) ───
# Old → New এ schema migrate করো
pg_dump -U postgres -h 172.16.93.140 \
    --schema-only mydb | \
    psql -U postgres -h 172.16.93.143 -d mydb

# ─── Step 4: Old server এ Publication তৈরি করো ───
# (PG 14 এ)
psql -U postgres -h 172.16.93.140 -d mydb -c "
CREATE PUBLICATION upgrade_pub FOR ALL TABLES;"

# pg_hba.conf এ New server allow করো:
# host mydb replicator 172.16.93.143/32 scram-sha-256

# ─── Step 5: New server এ Subscription তৈরি করো ───
# (PG 16 এ)
psql -U postgres -h 172.16.93.143 -d mydb -c "
CREATE SUBSCRIPTION upgrade_sub
    CONNECTION 'host=172.16.93.140 port=5432 dbname=mydb
                user=replicator password=ReplicaPass@2024!'
    PUBLICATION upgrade_pub;"

# Initial sync শুরু হবে (existing data copy হবে)
# বড় database হলে অনেক সময় লাগতে পারে

# ─── Step 6: Sync monitor করো ───
# New server এ:
psql -U postgres -h 172.16.93.143 -d mydb -c "
SELECT subname, received_lsn, latest_end_lsn,
    latest_end_lsn = received_lsn AS is_caught_up
FROM pg_stat_subscription;"

# Old server এ lag দেখো:
psql -U postgres -h 172.16.93.140 -d mydb -c "
SELECT slot_name, active,
    pg_size_pretty(pg_wal_lsn_diff(
        pg_current_wal_lsn(), confirmed_flush_lsn
    )) AS lag
FROM pg_replication_slots
WHERE slot_name LIKE 'upgrade%';"

# ─── Step 7: Cutover (maintenance window) ───
# Lag যখন near-zero:

# 7a. Application → read-only mode করো (বা connection close করো)

# 7b. Old server এ সব active connection check করো:
psql -U postgres -h 172.16.93.140 -c "
SELECT count(*) FROM pg_stat_activity WHERE datname='mydb' AND state='active';"
# 0 হলে proceed করো

# 7c. Final lag check করো (0 হওয়া উচিত):
psql -U postgres -h 172.16.93.143 -d mydb -c "
SELECT now() - latest_end_time AS lag FROM pg_stat_subscription;"

# 7d. Old server এ sequence values নাও এবং New server এ set করো:
psql -U postgres -h 172.16.93.140 -d mydb -c "
SELECT 'SELECT setval('''||schemaname||'.'||sequencename||''','||
    last_value||');'
FROM pg_sequences;" > /tmp/sequences.sql

psql -U postgres -h 172.16.93.143 -d mydb -f /tmp/sequences.sql

# 7e. Application → New server (PG 16) এ point করো
# Config/DNS update করো

# ─── Step 8: Cleanup ───
# New server এ subscription drop করো:
psql -U postgres -h 172.16.93.143 -d mydb -c "DROP SUBSCRIPTION upgrade_sub;"

# Old server এ publication drop করো:
psql -U postgres -h 172.16.93.140 -d mydb -c "DROP PUBLICATION upgrade_pub;"

# Old server কয়েকদিন রাখো (emergency rollback এর জন্য), তারপর decommission
```

---

## 13.5 Pre-upgrade Checklist এবং Common Issues

```bash
# ─── Pre-upgrade checklist script ───
cat > /usr/local/bin/pre_upgrade_check.sh << 'SCRIPT'
#!/bin/bash
echo "=== Pre-Upgrade Checklist ==="
echo "Old version: $(psql -U postgres -t -A -c 'SELECT version();')"

echo ""
echo "--- Extensions (manually reinstall needed on new server) ---"
psql -U postgres -t -c "SELECT name, default_version FROM pg_available_extensions WHERE installed_version IS NOT NULL;"

echo ""
echo "--- Deprecated features check ---"
psql -U postgres -t -c "SELECT name, setting FROM pg_settings WHERE name IN ('standard_conforming_strings','escape_string_warning','ssl_renegotiation_limit');"

echo ""
echo "--- Database sizes ---"
psql -U postgres -t -c "SELECT datname, pg_size_pretty(pg_database_size(datname)) FROM pg_database WHERE datname NOT IN ('template0','template1') ORDER BY pg_database_size(datname) DESC;"

echo ""
echo "--- Custom functions count ---"
psql -U postgres -t -c "SELECT count(*) AS custom_functions FROM pg_proc WHERE pronamespace='public'::regnamespace;"

echo ""
echo "--- Active connections ---"
psql -U postgres -t -c "SELECT count(*) FROM pg_stat_activity WHERE state='active';"

echo ""
echo "--- Replication slots (must be inactive before upgrade) ---"
psql -U postgres -t -c "SELECT slot_name, active, restart_lsn FROM pg_replication_slots;"
SCRIPT
chmod +x /usr/local/bin/pre_upgrade_check.sh
sudo -u postgres /usr/local/bin/pre_upgrade_check.sh
```

**Common Issues এবং Fix:**

```bash
# Issue 1: Extension এর নতুন version নেই new server এ
# Fix: Install করো
sudo dnf install -y pgcrypto_16 postgis35_16

# Issue 2: Encoding mismatch
# Old: SQL_ASCII → New: UTF8
# Fix: pg_dump --encoding=UTF8 ব্যবহার করো
pg_dump -U postgres --encoding=UTF8 mydb > /backup/mydb_utf8.sql

# Issue 3: pg_upgrade এ "could not read symbolic link" error
# Fix: SELinux context ঠিক করো
sudo restorecon -Rv /var/lib/pgsql/16/data/

# Issue 4: Logical replication এ sequence sync হয় না
# Fix: Manually sequence set করো (Step 7d উপরে দেখো)

# Issue 5: pg_upgrade পরে query planner slow
# Fix: Statistics rebuild করো
vacuumdb --all --analyze-in-stages -U postgres

# Issue 6: contrib functions missing (uuid-ossp, pgcrypto)
# Fix: New DB তে extension create করো
psql -U postgres -d mydb -c "CREATE EXTENSION IF NOT EXISTS pgcrypto;"
psql -U postgres -d mydb -c "CREATE EXTENSION IF NOT EXISTS \"uuid-ossp\";"
```

**Upgrade Method Selection:**

```
Database size:
  < 50GB:   pg_dump + pg_restore (simple, safe)
  50GB-1TB: pg_upgrade --link (fast, minimal downtime)
  > 1TB:    Logical replication (zero downtime) অথবা pg_upgrade --link

Downtime tolerance:
  Hours ok:     pg_dump + pg_restore
  Minutes ok:   pg_upgrade
  Seconds only: Logical replication

Rollback requirement:
  Easy rollback: pg_dump (old server intact) বা Logical replication
  Hard rollback: pg_upgrade --link (shared files)
```


---

# 14. Extensions — PostgreSQL Ecosystem

## 14.1 Extension কী এবং কীভাবে কাজ করে

```sql
-- Available extensions দেখো
SELECT name, default_version, installed_version, comment
FROM pg_available_extensions
ORDER BY name;

-- Install করো
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Installed extensions দেখো
\dx    -- psql shortcut
SELECT name, default_version, installed_version
FROM pg_available_extensions
WHERE installed_version IS NOT NULL;

-- Upgrade করো
ALTER EXTENSION pgcrypto UPDATE;

-- Remove করো
DROP EXTENSION pgcrypto;
```

---

## 14.2 pg_cron — Database থেকে Scheduled Jobs

OS cron এর বদলে database এর ভেতরে scheduled jobs চালাও।

```bash
# Install
sudo dnf install -y pg_cron_16
```

```ini
# postgresql.conf
shared_preload_libraries = 'pg_cron'
cron.database_name = 'postgres'   # pg_cron কোন DB তে run হবে
```

```sql
-- Extension load করো (postgres database এ)
CREATE EXTENSION pg_cron;

-- ─── Job schedule করো ───
-- প্রতি রাত ২টায় পুরনো logs delete করো
SELECT cron.schedule(
    'cleanup-old-logs',           -- job name
    '0 2 * * *',                  -- cron expression (every day 2am)
    $$DELETE FROM logs WHERE created_at < NOW() - INTERVAL '90 days'$$
);

-- প্রতি ঘন্টায় materialized view refresh করো
SELECT cron.schedule(
    'refresh-summary',
    '0 * * * *',
    $$REFRESH MATERIALIZED VIEW CONCURRENTLY daily_sales_summary$$
);

-- প্রতি সোমবার সকাল ৬টায় VACUUM ANALYZE চালাও
SELECT cron.schedule(
    'weekly-vacuum',
    '0 6 * * 1',
    $$VACUUM ANALYZE orders$$
);

-- প্রতি ৫ মিনিটে expired sessions delete করো
SELECT cron.schedule(
    'clear-sessions',
    '*/5 * * * *',
    $$DELETE FROM sessions WHERE expires_at < NOW()$$
);

-- ─── Job management ───
-- সব jobs দেখো
SELECT jobid, jobname, schedule, command, active
FROM cron.job
ORDER BY jobname;

-- Job disable করো
UPDATE cron.job SET active = false WHERE jobname = 'weekly-vacuum';

-- Job delete করো
SELECT cron.unschedule('cleanup-old-logs');

-- Job execution history দেখো
SELECT jobid, start_time, end_time, return_message, status
FROM cron.job_run_details
ORDER BY start_time DESC
LIMIT 20;

-- Failed jobs দেখো
SELECT jobid, start_time, return_message
FROM cron.job_run_details
WHERE status = 'failed'
ORDER BY start_time DESC;
```

---

## 14.3 pg_partman — Automatic Partition Management

Manual partition তৈরি করা ঝামেলা। pg_partman automatically করে।

```bash
sudo dnf install -y pg_partman_16
```

```sql
CREATE EXTENSION pg_partman;

-- ─── Time-based automatic partitioning ───
-- Parent table তৈরি করো
CREATE TABLE events (
    id         BIGINT NOT NULL,
    event_type TEXT,
    payload    JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- pg_partman দিয়ে configure করো
SELECT partman.create_parent(
    p_parent_table  => 'public.events',
    p_control       => 'created_at',          -- partition column
    p_type          => 'range',
    p_interval      => 'monthly',             -- monthly partitions
    p_premake       => 3                      -- future 3 months আগে তৈরি করো
);

-- automatic maintenance setup করো (cron বা pg_cron দিয়ে)
-- প্রতি ঘন্টায়:
SELECT cron.schedule('partman-maintenance', '0 * * * *',
    $$SELECT partman.run_maintenance()$$);

-- এখন ব্যবহার করো — partition automatically তৈরি হবে
INSERT INTO events (id, event_type, created_at)
VALUES (1, 'login', '2024-03-08 14:30:00');

-- Partitions দেখো
SELECT tablename, pg_size_pretty(pg_relation_size(tablename::regclass)) AS size
FROM pg_tables
WHERE tablename LIKE 'events_%'
ORDER BY tablename;

-- Old partitions automatically drop করো
UPDATE partman.part_config
SET retention = '12 months',        -- 12 মাসের বেশি পুরনো drop করো
    retention_keep_table = false     -- table drop করো
WHERE parent_table = 'public.events';
```

---

## 14.4 hypopg — Hypothetical Indexes

Real index না বানিয়ে দেখো — যদি এই index থাকতো, planner কি use করতো?

```bash
sudo dnf install -y hypopg_16
```

```sql
CREATE EXTENSION hypopg;

-- ─── Hypothetical index তৈরি করো ───
-- Slow query আছে:
EXPLAIN SELECT * FROM orders WHERE user_id = 5 AND status = 'pending';
-- Seq Scan — slow

-- Real index বানানো ছাড়াই test করো:
SELECT hypopg_create_index('CREATE INDEX ON orders(user_id, status)');
-- Returns: (1, btree_orders_user_id_status_hypothetical)

-- এখন EXPLAIN করো (hypopg এর virtual index use হবে কিনা দেখো):
EXPLAIN SELECT * FROM orders WHERE user_id = 5 AND status = 'pending';
-- Index Scan using btree_orders_user_id_status_hypothetical দেখালে ✅
-- এই index real করলে performance ভালো হবে

-- তারপর real index বানাও:
CREATE INDEX idx_orders_user_status ON orders(user_id, status);

-- Hypothetical indexes সরাও:
SELECT hypopg_reset();

-- ─── Useful workflow ───
-- Slow query list থেকে একটা নাও
-- hypopg দিয়ে বিভিন্ন index combination test করো
-- যেটাতে Index Scan আসে সেটা real index বানাও
-- Production এ CREATE INDEX CONCURRENTLY ব্যবহার করো
```

---

## 14.5 auto_explain — Slow Query এর Plan Auto-capture

Slow query হলে automatically execution plan log করো।

```ini
# postgresql.conf
shared_preload_libraries = 'auto_explain'

auto_explain.log_min_duration = 1000    # 1 second এর বেশি হলে log করো
auto_explain.log_analyze      = true    # ANALYZE সহ (actual rows, timing)
auto_explain.log_buffers      = true    # Buffer hit/miss দেখাও
auto_explain.log_timing       = true    # Per-node timing
auto_explain.log_nested_statements = true  # Nested function calls ও
auto_explain.log_verbose      = false   # Extra verbose (optional)
auto_explain.log_format       = text    # text বা json
```

```sql
-- Test করো (session level এ enable করা যায়)
LOAD 'auto_explain';
SET auto_explain.log_min_duration = 0;   -- সব query log করো (debug)
SET auto_explain.log_analyze = true;

SELECT pg_sleep(0.1);  -- dummy slow query

-- Log দেখো:
-- duration: 100.xxx ms  plan:
-- Query Text: SELECT pg_sleep(0.1)
-- Result  (cost=0.00..0.01 rows=1 width=4)
--           (actual time=100.xxx..100.xxx rows=1 loops=1)

-- Production এ: log_min_duration = 5000 (5 seconds)
-- pg_stat_monitor দিলে আরো ভালো (plan capture built-in)
```

---

## 14.6 pg_trgm — Trigram Similarity Search

LIKE '%keyword%' query কে fast করো।

```sql
CREATE EXTENSION pg_trgm;

-- ─── Index তৈরি করো ───
CREATE INDEX idx_products_trgm ON products USING GIN(name gin_trgm_ops);
CREATE INDEX idx_users_trgm ON users USING GIN(email gin_trgm_ops);

-- ─── Fast LIKE/ILIKE এখন ───
EXPLAIN ANALYZE SELECT * FROM products WHERE name ILIKE '%phone%';
-- Bitmap Index Scan using idx_products_trgm ← fast!

-- ─── Similarity search ───
SELECT name, similarity(name, 'iphone') AS score
FROM products
WHERE name % 'iphone'      -- % = similarity threshold (default 0.3)
ORDER BY score DESC
LIMIT 10;
-- Typo tolerant search: 'ifone', 'iphon' → match করবে

-- ─── Threshold adjust করো ───
SELECT set_limit(0.4);     -- similarity threshold বাড়াও (কম false positive)
SELECT show_limit();       -- current threshold দেখো

-- ─── Word similarity ───
SELECT word_similarity('smartphone android', 'android phone');
-- Partial match score
```

---

## 14.7 PostGIS — Geographic Data

Location-based data store এবং query করো।

```bash
sudo dnf install -y postgis35_16
```

```sql
CREATE EXTENSION postgis;

-- ─── Table তৈরি করো ───
CREATE TABLE places (
    id    SERIAL PRIMARY KEY,
    name  TEXT,
    geom  GEOMETRY(Point, 4326)  -- 4326 = WGS84 (GPS coordinates)
);

-- ─── Data insert করো ───
INSERT INTO places (name, geom) VALUES
('Dhaka', ST_SetSRID(ST_MakePoint(90.4125, 23.8103), 4326)),
('Chittagong', ST_SetSRID(ST_MakePoint(91.8123, 22.3569), 4326)),
('Sylhet', ST_SetSRID(ST_MakePoint(91.8777, 24.8949), 4326));

-- ─── Distance calculate করো ───
SELECT a.name, b.name,
    ST_Distance(
        a.geom::geography,
        b.geom::geography
    ) / 1000 AS distance_km
FROM places a, places b
WHERE a.id < b.id
ORDER BY distance_km;

-- ─── Nearby places খোঁজো (radius search) ───
-- Dhaka থেকে 200km এর মধ্যে:
SELECT name,
    ST_Distance(geom::geography,
        ST_SetSRID(ST_MakePoint(90.4125, 23.8103), 4326)::geography
    ) / 1000 AS km
FROM places
WHERE ST_DWithin(
    geom::geography,
    ST_SetSRID(ST_MakePoint(90.4125, 23.8103), 4326)::geography,
    200000  -- 200,000 meters = 200km
)
ORDER BY km;

-- ─── GiST Index (spatial queries fast করো) ───
CREATE INDEX idx_places_geom ON places USING GIST(geom);
-- ST_DWithin, ST_Within, ST_Intersects এ index use করবে
```

---

## 14.8 TimescaleDB — Time-Series Data

Metrics, IoT, financial data এর জন্য।

```bash
sudo dnf install -y timescaledb-2-postgresql-16
# postgresql.conf এ:
# shared_preload_libraries = 'timescaledb'
sudo systemctl restart postgresql-16
```

```sql
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- ─── Hypertable তৈরি করো ───
-- Regular table তৈরি করো
CREATE TABLE sensor_data (
    time        TIMESTAMPTZ NOT NULL,
    sensor_id   INTEGER,
    temperature DOUBLE PRECISION,
    humidity    DOUBLE PRECISION
);

-- Hypertable এ convert করো (automatically time-based partitioned)
SELECT create_hypertable('sensor_data', 'time');

-- ─── Data insert ও query ───
INSERT INTO sensor_data (time, sensor_id, temperature, humidity)
SELECT
    NOW() - (random() * INTERVAL '30 days'),
    (random() * 10)::int,
    20 + random() * 15,
    40 + random() * 40
FROM generate_series(1, 1000000);

-- Time-bucket aggregate (TimescaleDB এর killer feature)
SELECT time_bucket('1 hour', time) AS hour,
    sensor_id,
    AVG(temperature) AS avg_temp,
    MAX(temperature) AS max_temp,
    MIN(temperature) AS min_temp
FROM sensor_data
WHERE time > NOW() - INTERVAL '7 days'
GROUP BY hour, sensor_id
ORDER BY hour DESC;

-- ─── Compression (old data compress করো) ───
ALTER TABLE sensor_data SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'sensor_id'
);

-- 30 দিনের বেশি পুরনো data compress করো
SELECT add_compression_policy('sensor_data', INTERVAL '30 days');

-- ─── Data retention (old data automatically delete) ───
SELECT add_retention_policy('sensor_data', INTERVAL '1 year');
```

---

## 14.9 pg_stat_statements এবং Extensions Setup Script

```bash
# ─── Production Extensions Setup Script ───
cat > /usr/local/bin/setup_extensions.sh << 'SCRIPT'
#!/bin/bash
DB=${1:-mydb}
echo "Setting up extensions for database: $DB"

psql -U postgres -d $DB << 'SQL'
-- Query Performance
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;  -- query tracking

-- Crypto
CREATE EXTENSION IF NOT EXISTS pgcrypto;            -- encryption, hashing
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";         -- UUID generation

-- Text Search
CREATE EXTENSION IF NOT EXISTS pg_trgm;             -- fuzzy search

-- Utility
CREATE EXTENSION IF NOT EXISTS tablefunc;           -- crosstab, pivot
CREATE EXTENSION IF NOT EXISTS unaccent;            -- accent-insensitive search
CREATE EXTENSION IF NOT EXISTS hstore;              -- key-value store
CREATE EXTENSION IF NOT EXISTS ltree;               -- hierarchical tree data

-- Statistics
CREATE EXTENSION IF NOT EXISTS pgstattuple;         -- table/index bloat

-- Verify
\dx
SQL

echo "Extensions setup complete!"
SCRIPT
chmod +x /usr/local/bin/setup_extensions.sh
sudo -u postgres /usr/local/bin/setup_extensions.sh mydb
```

---

## 14.10 Extension Quick Reference

| Extension | কাজ | Install |
|---|---|---|
| `pg_stat_statements` | Query performance tracking | Built-in |
| `pgcrypto` | Encryption, hashing (bcrypt) | Built-in contrib |
| `uuid-ossp` | UUID generation | Built-in contrib |
| `pg_trgm` | Fuzzy/LIKE search fast | Built-in contrib |
| `pgstattuple` | Table/index bloat measure | Built-in contrib |
| `tablefunc` | Pivot/crosstab queries | Built-in contrib |
| `hstore` | Key-value in a column | Built-in contrib |
| `ltree` | Hierarchical tree data | Built-in contrib |
| `pg_cron` | Database-level cron jobs | External install |
| `pg_partman` | Automatic partition management | External install |
| `hypopg` | Hypothetical index testing | External install |
| `auto_explain` | Slow query plan auto-log | Built-in |
| `pg_repack` | Online table reorganization | External install |
| `PostGIS` | Geographic/spatial data | External install |
| `TimescaleDB` | Time-series optimization | External install |
| `pg_stat_monitor` | Enhanced query analytics | Percona/External |

```sql
-- সব available extensions দেখো
SELECT name, comment
FROM pg_available_extensions
ORDER BY name;

-- Specific extension এর functions দেখো
\dx+ pgcrypto    -- psql এ
-- অথবা:
SELECT proname, proargtypes::text
FROM pg_proc
WHERE pronamespace = (SELECT oid FROM pg_namespace WHERE nspname = 'public')
  AND prosrc LIKE '%crypto%';
```


---

# 15. Troubleshooting Scenarios এবং Schema Management

## 15.1 "Database Suddenly Slow" — Systematic Diagnosis

```sql
-- ─── Step 1: কী হচ্ছে এখন? ───
SELECT pid, usename, application_name,
    state, wait_event_type, wait_event,
    now() - query_start AS query_duration,
    LEFT(query, 100) AS query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY query_duration DESC NULLS LAST;

-- অনেক queries waiting → কী wait করছে?
SELECT wait_event_type, wait_event, COUNT(*)
FROM pg_stat_activity
WHERE wait_event IS NOT NULL
GROUP BY wait_event_type, wait_event
ORDER BY COUNT DESC;
```

```bash
# ─── Step 2: OS level কী হচ্ছে? ───
# CPU
top -bn1 | grep "postgres" | head -5
# High CPU → expensive queries চলছে

# Disk I/O
iostat -x 1 3
# %iowait বেশি → disk bottleneck

# Memory
free -h
# Swap ব্যবহার হচ্ছে? → RAM কম বা shared_buffers বেশি
```

```sql
-- ─── Step 3: Lock আছে কিনা দেখো ───
SELECT
    blocked.pid         AS blocked_pid,
    blocked.query       AS blocked_query,
    blocking.pid        AS blocking_pid,
    blocking.query      AS blocking_query,
    now() - blocked.xact_start AS waiting_for
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
    ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE cardinality(pg_blocking_pids(blocked.pid)) > 0;
-- Lock আছে → blocking query kill করো

-- ─── Step 4: কোন queries সবচেয়ে বেশি load নিচ্ছে? ───
SELECT LEFT(query, 80) AS query, calls,
    ROUND(total_exec_time::numeric, 0) AS total_ms,
    ROUND(mean_exec_time::numeric, 0) AS avg_ms
FROM pg_stat_statements
ORDER BY total_exec_time DESC LIMIT 10;

-- ─── Step 5: Buffer hit rate দেখো ───
SELECT ROUND(
    sum(heap_blks_hit) * 100.0 /
    NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0), 2
) AS hit_rate FROM pg_statio_user_tables;
-- Suddenly কমলে → workset RAM এ fit করছে না, বা cold cache

-- ─── Step 6: Table bloat দেখো ───
SELECT relname, n_dead_tup,
    ROUND(n_dead_tup*100.0/NULLIF(n_live_tup+n_dead_tup,0),1) AS dead_pct,
    last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC LIMIT 5;
-- Bloat বেশি → VACUUM চলেনি → autovacuum check করো
```

---

## 15.2 OOM Killer PostgreSQL Kill করলে

```bash
# OOM kill হয়েছে কিনা দেখো
sudo dmesg | grep -i "oom\|kill" | tail -20
sudo journalctl -k | grep -i "oom" | tail -20

# PostgreSQL log এ দেখো
sudo grep "FATAL\|server process.*was killed\|crash recovery" \
    /var/lib/pgsql/16/data/log/postgresql-$(date +%Y-%m-%d).log | tail -20

# Recovery হয়েছে কিনা check করো
psql -U postgres -c "SELECT pg_is_in_recovery();"
psql -U postgres -c "SELECT pg_last_xact_replay_timestamp();"
```

```bash
# Prevention:
# 1. vm.overcommit_memory = 2 set করো (OS এ)
echo "vm.overcommit_memory = 2" >> /etc/sysctl.d/99-postgresql.conf
echo "vm.overcommit_ratio = 80" >> /etc/sysctl.d/99-postgresql.conf
sysctl -p /etc/sysctl.d/99-postgresql.conf

# 2. OOM score adjust করো (PostgreSQL কে protect করো)
echo -1000 > /proc/$(pgrep -f "postgres: postmaster")/oom_score_adj

# Persistent করো (systemd service এ):
sudo mkdir -p /etc/systemd/system/postgresql-16.service.d/
sudo tee /etc/systemd/system/postgresql-16.service.d/oom.conf << 'EOF'
[Service]
OOMScoreAdjust=-1000
EOF
sudo systemctl daemon-reload

# 3. shared_buffers কমাও (যদি বেশি দেওয়া থাকে)
psql -U postgres -c "SHOW shared_buffers;"
# RAM এর 25% এর বেশি না
```

---

## 15.3 Data Corruption Detection এবং Recovery

```bash
# ─── Data checksums enable আছে কিনা ───
psql -U postgres -c "SHOW data_checksums;"
# on → corruption detect করতে পারবে
# off → enable করতে হলে (server offline থাকতে হবে):
sudo systemctl stop postgresql-16
sudo -u postgres /usr/pgsql-16/bin/pg_checksums \
    --enable --pgdata=/var/lib/pgsql/16/data
sudo systemctl start postgresql-16

# ─── Corruption check করো ───
sudo -u postgres /usr/pgsql-16/bin/pg_checksums \
    --check --pgdata=/var/lib/pgsql/16/data
# Bad checksums: 0 → all good
# Error থাকলে: কোন file corrupt তা দেখাবে

# ─── Corrupt table identify করো ───
# Log এ দেখো:
sudo grep "invalid page\|checksum\|corrupt" \
    /var/lib/pgsql/16/data/log/postgresql-*.log | tail -20

# ─── Corrupt page skip করে data নাও ───
# Last resort — corrupt pages skip করে যতটুকু পারা যায় data নাও:
psql -U postgres -d mydb -c "
SET zero_damaged_pages = on;  -- corrupt page কে zero দিয়ে replace করো
SELECT * FROM corrupted_table;
COPY (SELECT * FROM corrupted_table) TO '/tmp/recovered_data.csv' CSV;
"

# ─── Backup থেকে specific table restore ───
# pg_dump দিয়ে নতুন backup নাও আগে corrupt হয়নি এমন অবস্থায়
# তারপর corrupt table:
pg_restore -U postgres -d mydb -t corrupted_table /backup/mydb.dump
```

---

## 15.4 Connection Exhaustion (Too many connections)

```bash
# ─── Diagnose ───
psql -U postgres -c "
SELECT state, COUNT(*),
    ROUND(COUNT(*)*100.0/(SELECT setting::int FROM pg_settings WHERE name='max_connections'),1) AS pct
FROM pg_stat_activity
GROUP BY state
ORDER BY COUNT DESC;"

# কোন application সবচেয়ে বেশি connection নিচ্ছে?
psql -U postgres -c "
SELECT application_name, usename, client_addr, COUNT(*)
FROM pg_stat_activity
GROUP BY application_name, usename, client_addr
ORDER BY COUNT DESC LIMIT 10;"

# Idle connection কতক্ষণ ধরে idle?
psql -U postgres -c "
SELECT pid, now() - state_change AS idle_for,
    application_name, client_addr
FROM pg_stat_activity
WHERE state = 'idle'
ORDER BY idle_for DESC LIMIT 10;"
```

```bash
# ─── Emergency fix ───
# Idle connections kill করো
psql -U postgres -c "
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle'
  AND now() - state_change > INTERVAL '10 minutes'
  AND usename != 'postgres';"

# Permanent fix: pgBouncer install করো
sudo dnf install -y pgbouncer
# pgBouncer config করো (Section 10.8 দেখো)

# postgresql.conf এ timeout set করো
psql -U postgres -c "
ALTER SYSTEM SET idle_in_transaction_session_timeout = '5min';
ALTER SYSTEM SET tcp_keepalives_idle = 60;
SELECT pg_reload_conf();"
```

---

## 15.5 Schema Management Best Practices

```sql
-- ─── Schema তৈরি করো (application isolation) ───
CREATE SCHEMA app;          -- Application tables
CREATE SCHEMA audit;        -- Audit logs
CREATE SCHEMA reporting;    -- Reporting/analytics views
CREATE SCHEMA staging;      -- Import staging area

-- ─── search_path — Default schema set করো ───
-- Per session:
SET search_path TO app, public;

-- Per user (persistent):
ALTER USER appuser SET search_path TO app, public;

-- Per database (persistent):
ALTER DATABASE mydb SET search_path TO app, public;

-- ─── Schema access control ───
GRANT USAGE ON SCHEMA app TO appuser;
GRANT ALL ON ALL TABLES IN SCHEMA app TO appuser;
ALTER DEFAULT PRIVILEGES IN SCHEMA app
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO appuser;

-- Reporting user: read-only access
GRANT USAGE ON SCHEMA app, reporting TO reporter;
GRANT SELECT ON ALL TABLES IN SCHEMA app TO reporter;
ALTER DEFAULT PRIVILEGES IN SCHEMA app
    GRANT SELECT ON TABLES TO reporter;

-- ─── Tablespace — Data কোথায় store হবে ───
-- Different disk partition এ tablespace তৈরি করো
CREATE TABLESPACE fast_ssd LOCATION '/mnt/nvme/pg_data';
CREATE TABLESPACE slow_hdd LOCATION '/mnt/hdd/pg_archive';

-- Hot data → fast SSD
CREATE TABLE recent_orders (...) TABLESPACE fast_ssd;
CREATE INDEX idx_recent ON recent_orders(created_at) TABLESPACE fast_ssd;

-- Archive data → slow HDD
CREATE TABLE orders_2020 PARTITION OF orders_history
    FOR VALUES FROM ('2020-01-01') TO ('2021-01-01')
    TABLESPACE slow_hdd;

-- Tablespace list দেখো
\db    -- psql
SELECT spcname, pg_tablespace_location(oid), pg_size_pretty(pg_tablespace_size(oid))
FROM pg_tablespace;

-- ─── Table এর schema এবং tablespace দেখো ───
SELECT schemaname, tablename, tablespace
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY schemaname, tablename;
```

---

## 15.6 MySQL থেকে PostgreSQL Migration Gotchas

```sql
-- ─── সাধারণ MySQL → PostgreSQL differences ───

-- 1. AUTO_INCREMENT → IDENTITY বা SERIAL
-- MySQL:      id INT AUTO_INCREMENT PRIMARY KEY
-- PostgreSQL: id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY

-- 2. DATETIME → TIMESTAMPTZ
-- MySQL:      created_at DATETIME DEFAULT CURRENT_TIMESTAMP
-- PostgreSQL: created_at TIMESTAMPTZ DEFAULT NOW()

-- 3. TINYINT(1) → BOOLEAN
-- MySQL:      is_active TINYINT(1) DEFAULT 1
-- PostgreSQL: is_active BOOLEAN DEFAULT TRUE

-- 4. UNSIGNED → CHECK constraint
-- MySQL:      age TINYINT UNSIGNED
-- PostgreSQL: age SMALLINT CHECK (age >= 0)

-- 5. String quoting — MySQL backtick → PostgreSQL double-quote
-- MySQL:      SELECT `name` FROM `users`
-- PostgreSQL: SELECT "name" FROM "users"  -- অথবা lowercase name, quote লাগে না

-- 6. LIMIT syntax same, কিন্তু OFFSET আলাদা
-- MySQL:      LIMIT 10, 20     (offset=10, count=20)
-- PostgreSQL: LIMIT 20 OFFSET 10

-- 7. GROUP BY strict mode — PostgreSQL সব SELECT column GROUP BY তে চায়
-- MySQL (lenient): SELECT name, city, COUNT(*) FROM users GROUP BY name
-- PostgreSQL: SELECT name, city, COUNT(*) FROM users GROUP BY name, city
--             (বা city কে aggregate function এ রাখো)

-- 8. ISNULL() → IS NULL
-- MySQL:      WHERE ISNULL(email)
-- PostgreSQL: WHERE email IS NULL

-- 9. IFNULL() → COALESCE()
-- MySQL:      IFNULL(name, 'Unknown')
-- PostgreSQL: COALESCE(name, 'Unknown')

-- 10. String concat
-- MySQL:      CONCAT(first, ' ', last)
-- PostgreSQL: first || ' ' || last  (অথবা CONCAT ও কাজ করে)

-- 11. NOW() same, কিন্তু MySQL SYSDATE() ≠ PostgreSQL
-- PostgreSQL equivalent: clock_timestamp() (function call time)
--                       statement_timestamp() (query start time)
--                       transaction_timestamp() / NOW() (txn start time)

-- ─── Migration tool ───
-- pgloader: MySQL → PostgreSQL automatic migration
-- sudo dnf install -y pgloader
pgloader mysql://user:pass@host/mysqldb \
         postgresql://user:pass@host/pgdb

-- অথবা config file দিয়ে:
cat > /tmp/migrate.load << 'EOF'
LOAD DATABASE
    FROM    mysql://myuser:mypass@172.16.93.140/mydb
    INTO    postgresql://postgres:pgpass@172.16.93.140/mydb
WITH include no drop, create tables, create indexes,
     reset sequences, foreign keys
SET maintenance_work_mem to '256MB', work_mem to '64MB';
EOF
pgloader /tmp/migrate.load
```

---

## 15.7 Disk Full — Emergency Recovery

Disk full হলে PostgreSQL write করতে পারে না → crash বা hang করে।

```bash
# ─── Step 1: কতটুকু disk ব্যবহার হচ্ছে দেখো ───
df -h /var/lib/pgsql
# 100% used → এখনই action নিতে হবে!

# কোথায় space যাচ্ছে?
du -sh /var/lib/pgsql/16/data/*/  2>/dev/null | sort -rh | head -10
du -sh /var/lib/pgsql/16/data/pg_wal/   # WAL segments
du -sh /var/lib/pgsql/16/data/base/     # Data files
du -sh /archive/wal/                    # Archive

# ─── Step 2: সবচেয়ে বড় tables/indexes ───
psql -U postgres -c "
SELECT schemaname, relname AS name,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||relname)) AS size
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(schemaname||'.'||relname) DESC
LIMIT 10;"

# ─── Step 3: WAL জমে গেলে — inactive replication slot ───
psql -U postgres -c "
SELECT slot_name, active,
    pg_size_pretty(pg_wal_lsn_diff(
        pg_current_wal_lsn(), restart_lsn)) AS wal_retained
FROM pg_replication_slots;"
# inactive slot WAL জমাচ্ছে → drop করো (data loss হবে ওই replica তে)
psql -U postgres -c "SELECT pg_drop_replication_slot('old_slot_name');"

# ─── Step 4: Temp files পরিষ্কার করো ───
ls -lh /var/lib/pgsql/16/data/base/pgsql_tmp/ 2>/dev/null
# পুরনো temp files (server down থাকলে জমে)
sudo -u postgres find /var/lib/pgsql/16/data/base/pgsql_tmp/ \
    -name "pgsql_tmp*" -mmin +60 -delete

# ─── Step 5: Archive পুরনো গুলো delete করো ───
# pgBackRest retention দিয়ে:
pgbackrest --stanza=main expire
# অথবা manual (30 দিনের বেশি পুরনো):
find /archive/wal/ -name "*.gz" -mtime +30 -delete

# ─── Step 6: Bloated table compress করো ───
# pg_repack (online, lock ছাড়া):
pg_repack -h localhost -U postgres -d mydb -t large_table

# VACUUM FULL (lock নেয় কিন্তু space OS কে দেয়):
psql -U postgres -d mydb -c "VACUUM FULL large_table;"

# ─── Step 7: Emergency space তৈরি করো ───
# বড় table archive করো
psql -U postgres -d mydb -c "
CREATE TABLE orders_archive_2022 AS
SELECT * FROM orders WHERE created_at < '2023-01-01';
DELETE FROM orders WHERE created_at < '2023-01-01';
VACUUM orders;"

# ─── Step 8: Monitoring — disk usage alert ───
cat > /usr/local/bin/disk_alert.sh << 'SCRIPT'
#!/bin/bash
THRESHOLD=85
DISK_PCT=$(df /var/lib/pgsql | tail -1 | awk '{print $5}' | tr -d '%')
if [ "$DISK_PCT" -gt "$THRESHOLD" ]; then
    echo "ALERT: PostgreSQL disk usage ${DISK_PCT}% on $(hostname)" | \
        mail -s "CRITICAL: Disk Full Warning" dba@company.com
    logger "CRITICAL: PostgreSQL disk ${DISK_PCT}% full"
fi
SCRIPT
chmod +x /usr/local/bin/disk_alert.sh
# Cron: */10 * * * * /usr/local/bin/disk_alert.sh
```

---

## 15.8 PostgreSQL Start হচ্ছে না — Diagnosis

```bash
# ─── Step 1: Error দেখো ───
sudo systemctl status postgresql-16
# অথবা:
sudo journalctl -u postgresql-16 -n 50

# ─── Step 2: PostgreSQL log দেখো ───
sudo tail -50 /var/lib/pgsql/16/data/log/postgresql-$(date +%Y-%m-%d).log
# Common errors:
# "could not bind IPv4 address": port 5432 অন্য process use করছে
# "could not write to file pg_wal": disk full
# "database system identifier differs": wrong data directory
# "could not open file pg_filenode.map": data directory corrupt

# ─── Step 3: Port conflict ───
sudo ss -tnlp | grep 5432
# অন্য process 5432 use করলে:
sudo lsof -i :5432
# Kill করো অথবা postgresql.conf এ port change করো

# ─── Step 4: Data directory permission ───
ls -la /var/lib/pgsql/16/
ls -la /var/lib/pgsql/16/data/
# postgres:postgres হওয়া উচিত এবং 700 permission
sudo chown -R postgres:postgres /var/lib/pgsql/16/data/
sudo chmod 700 /var/lib/pgsql/16/data/

# ─── Step 5: pg_control corrupt/missing ───
# pg_control = cluster এর state file
ls -la /var/lib/pgsql/16/data/global/pg_control
# Missing বা corrupt হলে:
sudo -u postgres /usr/pgsql-16/bin/pg_resetwal -D /var/lib/pgsql/16/data/
# ⚠️ WARNING: শেষ resort! Data loss হতে পারে। Backup থেকে restore করা better।

# ─── Step 6: SELinux block করছে কিনা ───
sudo ausearch -m avc -ts recent | grep postgres
# Block হলে:
sudo setsebool -P httpd_can_network_connect_db 1
sudo restorecon -Rv /var/lib/pgsql/

# ─── Step 7: Manual start করে error দেখো ───
sudo -u postgres /usr/pgsql-16/bin/postgres \
    -D /var/lib/pgsql/16/data/ \
    -c config_file=/var/lib/pgsql/16/data/postgresql.conf
# Error সরাসরি terminal এ দেখাবে
```

---

## 15.9 Autovacuum এবং XID Wraparound Emergency

```sql
-- ─── XID Wraparound Risk Check ───
SELECT
    datname,
    age(datfrozenxid)                                       AS xid_age,
    2000000000 - age(datfrozenxid)                         AS transactions_left,
    CASE
        WHEN age(datfrozenxid) > 1800000000 THEN '🚨 EMERGENCY'
        WHEN age(datfrozenxid) > 1500000000 THEN '⚠️  CRITICAL'
        WHEN age(datfrozenxid) > 1000000000 THEN '!  WARNING'
        ELSE '✓  OK'
    END AS status
FROM pg_database
ORDER BY xid_age DESC;
```

```bash
# ─── Emergency VACUUM FREEZE ───
# XID age > 1.5 billion → জরুরি action!

# Step 1: Autovacuum কি চলছে?
psql -U postgres -c "
SELECT pid, now()-query_start AS duration,
    LEFT(query, 80) AS query
FROM pg_stat_activity
WHERE query LIKE '%autovacuum%' OR query LIKE '%vacuum%';"

# Step 2: কোন table সবচেয়ে পুরনো XID নিয়ে আছে?
psql -U postgres -d mydb -c "
SELECT relname,
    age(relfrozenxid) AS table_xid_age,
    pg_size_pretty(pg_relation_size(oid)) AS size
FROM pg_class
WHERE relkind = 'r'
ORDER BY age(relfrozenxid) DESC
LIMIT 10;"

# Step 3: Manual aggressive VACUUM FREEZE চালাও
# (autovacuum কে bypass করে)
psql -U postgres -d mydb -c "
VACUUM (FREEZE, ANALYZE, VERBOSE) orders;"

# সব table এ:
psql -U postgres -c "VACUUM FREEZE ANALYZE;" -d mydb
# অথবা:
sudo -u postgres vacuumdb --all --freeze --analyze-in-stages -U postgres

# Step 4: Autovacuum aggressiveness বাড়াও (runtime এ)
psql -U postgres -c "
ALTER SYSTEM SET autovacuum_vacuum_cost_delay = '0ms';
ALTER SYSTEM SET autovacuum_vacuum_cost_limit = '2000';
ALTER SYSTEM SET autovacuum_max_workers = '8';
SELECT pg_reload_conf();"
# Emergency শেষে reset করো:
# ALTER SYSTEM RESET autovacuum_vacuum_cost_delay;
# SELECT pg_reload_conf();
```

---

## 15.10 Production DBA Daily Checklist

```bash
cat > /usr/local/bin/pg_daily_check.sh << 'SCRIPT'
#!/bin/bash
# Production DBA Daily Checklist
# Cron: 0 9 * * * postgres /usr/local/bin/pg_daily_check.sh | mail -s "PG Daily Report" dba@company.com

PSQL="psql -U postgres -t -A"
HOST=$(hostname)
DATE=$(date +%Y-%m-%d)

echo "================================================================"
echo "  PostgreSQL Daily Health Report — $HOST — $DATE"
echo "================================================================"

# 1. Version এবং Uptime
echo ""
echo "[ SERVER INFO ]"
$PSQL -c "SELECT version();"
$PSQL -c "SELECT 'Uptime: '||date_trunc('second', now() - pg_postmaster_start_time());"

# 2. Database sizes
echo ""
echo "[ DATABASE SIZES ]"
$PSQL -c "SELECT datname, pg_size_pretty(pg_database_size(datname)) AS size FROM pg_database WHERE datname NOT IN ('template0','template1') ORDER BY pg_database_size(datname) DESC;"

# 3. Connection utilization
echo ""
echo "[ CONNECTIONS ]"
$PSQL -c "SELECT state, count(*) FROM pg_stat_activity GROUP BY state ORDER BY count DESC;"
CONN_PCT=$($PSQL -c "SELECT ROUND(count(*)*100.0/(SELECT setting::int FROM pg_settings WHERE name='max_connections'),1) FROM pg_stat_activity;")
echo "Utilization: ${CONN_PCT}%"

# 4. Buffer hit rate
echo ""
echo "[ BUFFER HIT RATE ]"
$PSQL -c "SELECT ROUND(sum(heap_blks_hit)*100.0/NULLIF(sum(heap_blks_hit)+sum(heap_blks_read),0),2)||'%' AS hit_rate FROM pg_statio_user_tables;"

# 5. Replication status
echo ""
echo "[ REPLICATION ]"
$PSQL -c "SELECT application_name, state, sync_state, replay_lag FROM pg_stat_replication;" 2>/dev/null || \
$PSQL -c "SELECT 'Standby lag: '||(now()-pg_last_xact_replay_timestamp())::text;" 2>/dev/null

# 6. XID Age (Wraparound risk)
echo ""
echo "[ XID WRAPAROUND RISK ]"
$PSQL -c "SELECT datname, age(datfrozenxid) AS xid_age FROM pg_database ORDER BY xid_age DESC LIMIT 3;"

# 7. Dead tuples (tables needing vacuum)
echo ""
echo "[ TABLES NEEDING VACUUM (>10% dead) ]"
$PSQL -c "SELECT relname, n_dead_tup, ROUND(n_dead_tup*100.0/NULLIF(n_live_tup+n_dead_tup,0),1)||'%' AS dead_pct FROM pg_stat_user_tables WHERE n_dead_tup*100.0/NULLIF(n_live_tup+n_dead_tup,0) > 10 ORDER BY n_dead_tup DESC LIMIT 5;"

# 8. Long running queries
echo ""
echo "[ LONG RUNNING QUERIES (>5min) ]"
$PSQL -c "SELECT pid, now()-query_start AS duration, LEFT(query,80) FROM pg_stat_activity WHERE now()-query_start > INTERVAL '5 min' AND state='active' ORDER BY duration DESC LIMIT 5;"

# 9. Archive status
echo ""
echo "[ WAL ARCHIVE ]"
$PSQL -c "SELECT last_archived_wal, last_archived_time, failed_count FROM pg_stat_archiver;"

# 10. Disk usage
echo ""
echo "[ DISK USAGE ]"
df -h /var/lib/pgsql | tail -1

# 11. Top 5 slow queries (yesterday's)
echo ""
echo "[ TOP 5 SLOWEST QUERIES ]"
$PSQL -c "SELECT LEFT(query,80), calls, ROUND(mean_exec_time::numeric,0)||'ms' AS avg_time FROM pg_stat_statements ORDER BY mean_exec_time DESC LIMIT 5;" 2>/dev/null

echo ""
echo "================================================================"
echo "  Report generated: $(date)"
echo "================================================================"
SCRIPT
chmod +x /usr/local/bin/pg_daily_check.sh
echo "Daily check script created!"
```


---

# 16. Ultimate Quick Reference

## 16.1 Installation — Rocky Linux 9

```bash
# ─── PostgreSQL 16 Official (PGDG) ───
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo dnf -qy module disable postgresql
sudo dnf install -y postgresql16-server postgresql16-contrib
sudo /usr/pgsql-16/bin/postgresql-16-setup initdb
sudo systemctl enable --now postgresql-16
sudo -u postgres psql -c "ALTER USER postgres PASSWORD 'StrongPass@2024!';"

# ─── Percona Distribution ───
sudo dnf install -y https://repo.percona.com/yum/percona-release-latest.noarch.rpm
sudo percona-release setup ppg-16
sudo dnf install -y percona-postgresql16-server percona-postgresql16-contrib \
    percona-pg_stat_monitor16 percona-pgbackrest percona-patroni
sudo /usr/pgsql-16/bin/postgresql-16-setup initdb
sudo systemctl enable --now postgresql-16
```

---

## 16.2 psql — সব দরকারি Commands

```bash
# Connect করো
psql -U postgres                          # local, postgres user
psql -U postgres -d mydb                  # specific database
psql -h 172.16.93.140 -U postgres -d mydb # remote
psql "postgresql://postgres:pass@host/db" # connection string
```

```
\l          সব database list
\c mydb     database switch
\dt         current schema এর tables
\dt *.*     সব schema এর tables
\d users    table structure + indexes
\di         indexes list
\dv         views list
\df         functions list
\ds         sequences list
\du         users/roles list
\dn         schemas list
\dp users   table privileges
\db         tablespaces
\dx         installed extensions
\x          expanded display toggle (wide rows এ useful)
\timing     query timing toggle
\pset null '(null)'   NULL display করো
\i file.sql SQL file execute করো
\e          external editor তে query লেখো
\o file.txt output file এ save করো
\copy       file import/export (client-side)
\! command  shell command execute করো
\q          quit
\?          psql help
\h SELECT   SQL help
\sf func    function definition দেখো
```

---

## 16.3 Database এবং User Management

```sql
-- Database
CREATE DATABASE mydb
    WITH OWNER = appuser
         ENCODING = 'UTF8'
         LC_COLLATE = 'en_US.UTF-8'
         LC_CTYPE = 'en_US.UTF-8'
         TEMPLATE = template0;

ALTER DATABASE mydb SET timezone = 'UTC';
DROP DATABASE IF EXISTS mydb;
SELECT pg_size_pretty(pg_database_size('mydb'));

-- User/Role
CREATE USER appuser PASSWORD 'Pass@2024!' VALID UNTIL '2025-12-31';
CREATE ROLE readonly_role;
GRANT readonly_role TO appuser;
ALTER USER appuser SUPERUSER;      -- superuser দাও
ALTER USER appuser NOSUPERUSER;    -- revoke
ALTER USER appuser PASSWORD 'NewPass@2024!';
ALTER USER appuser VALID UNTIL 'infinity';
DROP USER IF EXISTS appuser;

-- Privileges
GRANT CONNECT ON DATABASE mydb TO appuser;
GRANT USAGE ON SCHEMA public TO appuser;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO appuser;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO appuser;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO appuser;
REVOKE ALL ON ALL TABLES IN SCHEMA public FROM olduser;
```

---

## 16.4 Table Operations

```sql
-- Create
CREATE TABLE users (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    username    TEXT NOT NULL UNIQUE CHECK (length(username) BETWEEN 3 AND 50),
    email       TEXT NOT NULL UNIQUE CHECK (email ~* '^[^@]+@[^@]+$'),
    status      TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active','inactive')),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Alter
ALTER TABLE users ADD COLUMN phone TEXT;
ALTER TABLE users DROP COLUMN phone;
ALTER TABLE users ALTER COLUMN username TYPE VARCHAR(100);
ALTER TABLE users ALTER COLUMN status SET DEFAULT 'pending';
ALTER TABLE users ADD CONSTRAINT chk_phone CHECK (phone ~ '^\+?[0-9]{10,15}$');
ALTER TABLE users RENAME COLUMN username TO user_name;
ALTER TABLE users RENAME TO app_users;

-- Index
CREATE INDEX idx_email ON users(email);
CREATE UNIQUE INDEX idx_username ON users(username);
CREATE INDEX idx_status ON users(status) WHERE status != 'active';  -- partial
CREATE INDEX idx_lower_email ON users(LOWER(email));                -- expression
CREATE INDEX idx_meta ON users USING GIN(metadata);                 -- GIN
CREATE INDEX CONCURRENTLY idx_created ON users(created_at DESC);    -- no lock
DROP INDEX CONCURRENTLY idx_old;

-- Constraints
ALTER TABLE orders ADD FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE;
ALTER TABLE users ADD CONSTRAINT uq_email UNIQUE (email);
ALTER TABLE users DROP CONSTRAINT uq_email;
```

---

## 16.5 Query Patterns — Most Used

```sql
-- Pagination
-- Offset-based (simple কিন্তু slow for large offset)
SELECT * FROM orders ORDER BY id LIMIT 20 OFFSET 100;

-- Keyset (fast, production recommended)
SELECT * FROM orders WHERE id > 1000 ORDER BY id LIMIT 20;

-- Upsert
INSERT INTO users (id, email, name)
VALUES (5, 'alice@example.com', 'Alice')
ON CONFLICT (id) DO UPDATE
    SET email = EXCLUDED.email, name = EXCLUDED.name, updated_at = NOW();

-- Bulk insert
INSERT INTO logs (level, message) SELECT 'info', msg FROM staging_logs;
COPY orders FROM '/tmp/orders.csv' CSV HEADER;

-- Update with JOIN
UPDATE orders o SET status = 'vip'
FROM users u WHERE o.user_id = u.id AND u.tier = 'platinum';

-- Delete with JOIN
DELETE FROM sessions s
USING users u WHERE s.user_id = u.id AND u.status = 'banned';

-- Conditional aggregation
SELECT
    COUNT(*) AS total,
    COUNT(*) FILTER (WHERE status = 'completed') AS completed,
    SUM(total) FILTER (WHERE status = 'completed') AS revenue,
    ROUND(AVG(total) FILTER (WHERE status = 'completed'), 2) AS avg_order
FROM orders WHERE created_at > NOW() - INTERVAL '30 days';

-- Window functions
SELECT user_id, order_id, total,
    ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY total DESC) AS rank,
    SUM(total) OVER (PARTITION BY user_id) AS user_total,
    SUM(total) OVER (ORDER BY created_at) AS running_total
FROM orders;

-- CTE
WITH monthly_revenue AS (
    SELECT DATE_TRUNC('month', created_at) AS month,
        SUM(total) AS revenue
    FROM orders WHERE status = 'completed'
    GROUP BY 1
)
SELECT month, revenue,
    revenue - LAG(revenue) OVER (ORDER BY month) AS growth
FROM monthly_revenue
ORDER BY month;

-- LATERAL JOIN (per-row subquery)
SELECT u.id, u.name, recent.* FROM users u
CROSS JOIN LATERAL (
    SELECT id, total FROM orders WHERE user_id = u.id
    ORDER BY created_at DESC LIMIT 3
) recent;
```

---

## 16.6 Performance Diagnosis — One-liners

```sql
-- Buffer hit rate (99%+ হওয়া উচিত)
SELECT ROUND(sum(heap_blks_hit)*100.0/NULLIF(sum(heap_blks_hit)+sum(heap_blks_read),0),2)||'%' FROM pg_statio_user_tables;

-- Active queries (slow queries দেখো)
SELECT pid, now()-query_start AS age, state, LEFT(query,100) FROM pg_stat_activity WHERE state='active' ORDER BY age DESC;

-- Blocking queries
SELECT blocked.pid, LEFT(blocked.query,60), blocking.pid, LEFT(blocking.query,60) FROM pg_stat_activity blocked JOIN pg_stat_activity blocking ON blocking.pid=ANY(pg_blocking_pids(blocked.pid)) WHERE cardinality(pg_blocking_pids(blocked.pid))>0;

-- Top slow queries
SELECT LEFT(query,80), calls, ROUND(mean_exec_time::numeric,0)||'ms' FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 5;

-- Table sizes
SELECT relname, pg_size_pretty(pg_total_relation_size(oid)) FROM pg_class WHERE relkind='r' AND relnamespace='public'::regnamespace ORDER BY pg_total_relation_size(oid) DESC LIMIT 10;

-- Dead tuple % (10%+ = VACUUM দরকার)
SELECT relname, ROUND(n_dead_tup*100.0/NULLIF(n_live_tup+n_dead_tup,0),1)||'%' AS dead_pct FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 10;

-- XID wraparound age
SELECT datname, age(datfrozenxid) FROM pg_database ORDER BY age(datfrozenxid) DESC LIMIT 3;

-- Connection count by state
SELECT state, count(*) FROM pg_stat_activity GROUP BY state;

-- Replication lag (Primary তে)
SELECT application_name, pg_size_pretty(sent_lsn-replay_lsn) AS lag FROM pg_stat_replication;

-- Replication lag (Replica তে)
SELECT now()-pg_last_xact_replay_timestamp() AS lag;

-- Unused indexes
SELECT schemaname, tablename, indexname, pg_size_pretty(pg_relation_size(indexrelid)) FROM pg_stat_user_indexes WHERE idx_scan=0 AND indexname NOT LIKE '%pkey%' ORDER BY pg_relation_size(indexrelid) DESC;

-- Temp file usage
SELECT datname, temp_files, pg_size_pretty(temp_bytes) FROM pg_stat_database WHERE temp_files>0;

-- Checkpoint frequency
SELECT checkpoints_req, checkpoints_timed, ROUND(checkpoints_req*100.0/NULLIF(checkpoints_req+checkpoints_timed,0),1)||'%' AS forced_pct FROM pg_stat_bgwriter;

-- Archive status
SELECT last_archived_wal, now()-last_archived_time AS age, failed_count FROM pg_stat_archiver;
```

---

## 16.7 Maintenance Commands

```sql
-- VACUUM
VACUUM users;                            -- basic vacuum
VACUUM VERBOSE ANALYZE users;           -- verbose + analyze
VACUUM FREEZE users;                     -- force freeze (XID)
VACUUM FULL users;                       -- full rewrite (lock!)
VACUUM (VERBOSE, ANALYZE, FREEZE) users;

-- ANALYZE (statistics update)
ANALYZE;                                 -- সব tables
ANALYZE VERBOSE users;                   -- specific table
ALTER TABLE orders ALTER COLUMN user_id SET STATISTICS 500;
ANALYZE orders;                          -- then re-analyze

-- REINDEX
REINDEX INDEX idx_email;                 -- single index
REINDEX TABLE users;                     -- all indexes on table
REINDEX DATABASE mydb;                   -- all indexes
REINDEX INDEX CONCURRENTLY idx_email;   -- online (PG 12+)
REINDEX TABLE CONCURRENTLY users;       -- online (PG 12+)

-- CLUSTER (sort table by index order)
CLUSTER users USING idx_users_created_at;
CLUSTER users;                           -- last clustered index ব্যবহার করো

-- TRUNCATE
TRUNCATE TABLE temp_data;
TRUNCATE TABLE temp_data RESTART IDENTITY CASCADE;
```

```bash
# CLI maintenance
vacuumdb -U postgres --all --analyze        # সব DB vacuum+analyze
vacuumdb -U postgres -d mydb --full         # full vacuum
reindexdb -U postgres -d mydb               # reindex
pg_repack -U postgres -d mydb -t orders     # online repack (no lock)
```

---

## 16.8 Config Quick Changes

```sql
-- Reload (restart ছাড়া)
ALTER SYSTEM SET log_min_duration_statement = 500;
ALTER SYSTEM SET work_mem = '64MB';
ALTER SYSTEM SET autovacuum_max_workers = 5;
SELECT pg_reload_conf();

-- Session level (current session only)
SET work_mem = '256MB';
SET search_path TO myschema, public;
RESET work_mem;

-- Transaction level
BEGIN;
SET LOCAL work_mem = '1GB';   -- transaction শেষে reset
SELECT ... heavy aggregation ...
COMMIT;

-- Check pending restart
SELECT name, setting, pending_restart FROM pg_settings WHERE pending_restart;

-- View all non-default settings
SELECT name, setting, source FROM pg_settings
WHERE source NOT IN ('default','environment variable')
ORDER BY name;
```

---

## 16.9 Backup Quick Reference

```bash
# pg_dump
pg_dump -U postgres -Fc mydb > mydb.dump                    # compressed
pg_dump -U postgres -Fc -j4 -f /backup/mydb_dir/ mydb       # parallel
pg_dump -U postgres -t users mydb > users_only.sql          # single table
pg_dumpall -U postgres --globals-only > globals.sql          # roles + tablespaces

# pg_restore
pg_restore -U postgres -d mydb mydb.dump                    # restore
pg_restore -U postgres -d mydb -j4 /backup/mydb_dir/       # parallel
pg_restore -U postgres -d mydb -t users mydb.dump           # single table
pg_restore -U postgres -C -d postgres mydb.dump             # create+restore

# pgBackRest
pgbackrest --stanza=main --type=full backup                  # full
pgbackrest --stanza=main --type=diff backup                  # differential
pgbackrest --stanza=main --type=incr backup                  # incremental
pgbackrest --stanza=main info                                # backup info
pgbackrest --stanza=main restore                             # restore latest
pgbackrest --stanza=main --type=time --target="2024-03-08 14:00:00" restore  # PITR
pgbackrest --stanza=main verify                              # verify integrity
pgbackrest --stanza=main expire                              # cleanup old backups

# pg_basebackup
pg_basebackup -U replicator -h primary -D /data -Xs -P      # streaming WAL
pg_basebackup -U replicator -h primary -D /data -Xs -R -P   # standby setup
```

---

## 16.10 Replication Quick Reference

```sql
-- Primary এ
SELECT pg_current_wal_lsn();                              -- current WAL position
SELECT pg_walfile_name(pg_current_wal_lsn());             -- current WAL file
SELECT pg_switch_wal();                                   -- force WAL switch
SELECT * FROM pg_stat_replication;                        -- standby status
SELECT pg_create_physical_replication_slot('slot1');      -- create slot
SELECT pg_drop_replication_slot('slot1');                 -- drop slot
SELECT * FROM pg_replication_slots;                       -- slot status
CHECKPOINT;                                               -- force checkpoint

-- Standby এ
SELECT pg_is_in_recovery();                               -- t = standby
SELECT pg_last_wal_receive_lsn();                         -- received
SELECT pg_last_wal_replay_lsn();                          -- applied
SELECT now() - pg_last_xact_replay_timestamp();           -- lag
SELECT pg_promote();                                      -- promote to primary
SELECT pg_wal_replay_pause();                             -- pause replay
SELECT pg_wal_replay_resume();                            -- resume replay
SELECT * FROM pg_stat_wal_receiver;                       -- receiver status
```

```bash
# Patroni
patronictl -c /etc/patroni/patroni.yml list               # cluster status
patronictl -c /etc/patroni/patroni.yml switchover main    # graceful switchover
patronictl -c /etc/patroni/patroni.yml failover main      # emergency failover
patronictl -c /etc/patroni/patroni.yml reinit main node1  # re-clone node
patronictl -c /etc/patroni/patroni.yml pause main         # pause auto-failover
patronictl -c /etc/patroni/patroni.yml resume main        # resume
patronictl -c /etc/patroni/patroni.yml edit-config        # edit cluster config
patronictl -c /etc/patroni/patroni.yml history main       # failover history
```

---

## 16.11 Common Error Messages — কী মানে, কী করবে

| Error | মানে | Fix |
|---|---|---|
| `FATAL: role "user" does not exist` | User নেই | `CREATE USER user;` |
| `FATAL: database "db" does not exist` | Database নেই | `CREATE DATABASE db;` |
| `FATAL: password authentication failed` | Wrong password | Password reset করো |
| `no pg_hba.conf entry for host` | pg_hba.conf এ rule নেই | Rule যোগ করো, reload করো |
| `could not connect to server: Connection refused` | Server চলছে না বা port wrong | `systemctl start postgresql-16` |
| `too many connections` | max_connections পূর্ণ | pgBouncer ব্যবহার করো |
| `deadlock detected` | Circular lock | Application এ consistent lock order |
| `could not obtain lock on row` | FOR UPDATE NOWAIT fail | Retry logic যোগ করো |
| `relation "table" does not exist` | Table নেই বা wrong schema | `SET search_path` বা schema qualify করো |
| `duplicate key value violates unique constraint` | Unique constraint violation | Upsert বা check-before-insert |
| `integer out of range` | INT overflow | BIGINT এ upgrade করো |
| `invalid byte sequence for encoding "UTF8"` | Encoding mismatch | `--encoding=UTF8` দিয়ে dump/restore |
| `WAL segment ... has already been removed` | Replica lag বেশি | `wal_keep_size` বাড়াও বা replication slot |
| `canceling statement due to conflict with recovery` | Standby query conflict | `hot_standby_feedback = on` |
| `could not open file "pg_control"` | Data directory corrupt | Backup থেকে restore করো |
| `database system identifier differs` | Wrong data directory | সঠিক `$PGDATA` point করো |
| `ERROR: syntax error at or near "...` | SQL syntax ভুল | Query check করো |
| `ERROR: column "x" of relation "y" does not exist` | Column নেই | `\d tablename` দিয়ে structure দেখো |
| `lock timeout` | Lock নেওয়ার আগে timeout | `lock_timeout` value বাড়াও বা blocking query kill করো |
| `statement timeout` | Query too long | `statement_timeout` বাড়াও বা query optimize করো |
| `out of shared memory` | Shared memory শেষ | `max_locks_per_transaction` বাড়াও |
| `could not write to file "pg_wal"` | Disk full | Disk space তৈরি করো |
| `autovacuum: found orphan temp table` | Crashed session এর temp table | Normal, autovacuum cleanup করবে |
| `terminating connection due to idle-in-transaction timeout` | Idle txn timeout | Application এ txn management fix করো |

---

*PostgreSQL 16 | Rocky Linux 9 | Complete DBA Reference*
*Architecture · Configuration · Data Types · Indexes · Transactions · Query Optimization*
*Backup & Recovery · Monitoring · Security · HA & Replication · Percona*
*PL/pgSQL · Version Upgrade · Extensions · Troubleshooting*

---

# 17. Lock Contention — Deep Dive

## 17.1 Lock কেন Production এর সবচেয়ে বড় সমস্যা

```
Lock ছাড়া concurrent access এ data corrupt হবে।
কিন্তু lock বেশি হলে:
  ① Queries queue তে আটকে থাকে
  ② Application timeout করে
  ③ Connection pool exhausted হয়
  ④ Cascading failure — পুরো application down

Real scenario:
  DBA: ALTER TABLE orders ADD COLUMN notes TEXT;
  → ACCESS EXCLUSIVE lock নেয়
  → চলমান সব SELECT ও block হয়
  → Connection pool ভরে যায়
  → Application 503 দেয়
  → DBA panic করে CTRL+C
  → Lock release হয়, কিন্তু ক্ষতি হয়ে গেছে
```

---

## 17.2 PostgreSQL Lock Hierarchy — সম্পূর্ণ চিত্র

```
Lock Strength (weak → strong):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. ACCESS SHARE (AS)
   কে নেয়: SELECT
   Blocks: শুধু ACCESS EXCLUSIVE
   মানে: "আমি পড়ছি, কেউ drop/truncate করো না"

2. ROW SHARE (RS)
   কে নেয়: SELECT FOR UPDATE, SELECT FOR SHARE
   Blocks: EXCLUSIVE, ACCESS EXCLUSIVE

3. ROW EXCLUSIVE (RX)
   কে নেয়: INSERT, UPDATE, DELETE
   Blocks: SHARE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE
   মানে: "আমি data change করছি"

4. SHARE UPDATE EXCLUSIVE (SUE)
   কে নেয়: VACUUM, ANALYZE, CREATE INDEX CONCURRENTLY
   Blocks: নিজের সাথে, SHARE, EXCLUSIVE, ACCESS EXCLUSIVE
   মানে: "Table structure ঠিক রাখো, কিন্তু read/write চলতে পারে"

5. SHARE (S)
   কে নেয়: CREATE INDEX (non-concurrent)
   Blocks: ROW EXCLUSIVE এবং তার উপরে
   ⚠️ DML (INSERT/UPDATE/DELETE) block করে!

6. SHARE ROW EXCLUSIVE (SRX)
   কে নেয়: CREATE TRIGGER, some ALTER TABLE
   Blocks: ROW EXCLUSIVE এবং তার উপরে

7. EXCLUSIVE (E)
   কে নেয়: REFRESH MATERIALIZED VIEW (non-concurrent)
   Blocks: প্রায় সব (ACCESS SHARE ছাড়া)

8. ACCESS EXCLUSIVE (AE) ← সবচেয়ে dangerous
   কে নেয়: DROP TABLE, TRUNCATE, VACUUM FULL,
             ALTER TABLE, REINDEX, LOCK TABLE
   Blocks: সব — SELECT ও block হয়!
   ⚠️ Production এ এটা নিলে সব query block!
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**Lock Compatibility Matrix:**
```
         AS  RS  RX  SUE  S  SRX  E   AE
AS        ✅  ✅  ✅   ✅  ✅   ✅  ✅   ❌
RS        ✅  ✅  ✅   ✅  ✅   ✅  ❌   ❌
RX        ✅  ✅  ✅   ✅  ❌   ❌  ❌   ❌
SUE       ✅  ✅  ✅   ❌  ❌   ❌  ❌   ❌
S         ✅  ✅  ❌   ❌  ✅   ❌  ❌   ❌
SRX       ✅  ✅  ❌   ❌  ❌   ❌  ❌   ❌
E         ✅  ❌  ❌   ❌  ❌   ❌  ❌   ❌
AE        ❌  ❌  ❌   ❌  ❌   ❌  ❌   ❌
✅ = compatible (একসাথে চলতে পারে)
❌ = conflict (একটা অন্যটাকে block করে)
```

---

## 17.3 কোন DDL কোন Lock নেয়

Production DBA এর জন্য সবচেয়ে গুরুত্বপূর্ণ table:

```sql
-- এই query দিয়ে যেকোনো statement এর lock দেখো
SELECT object_identity, mode
FROM pg_locks l
JOIN pg_stat_activity a ON l.pid = a.pid
WHERE a.query = 'YOUR_QUERY_HERE';
```

| Operation | Lock | DML Block? | SELECT Block? |
|---|---|---|---|
| `SELECT` | ACCESS SHARE | না | না |
| `INSERT/UPDATE/DELETE` | ROW EXCLUSIVE | না (নিজেদের মধ্যে) | না |
| `SELECT FOR UPDATE` | ROW SHARE | না | না |
| `CREATE INDEX` | SHARE | **হ্যাঁ!** | না |
| `CREATE INDEX CONCURRENTLY` | SUE | না | না |
| `DROP INDEX` | ACCESS EXCLUSIVE | **হ্যাঁ!** | **হ্যাঁ!** |
| `DROP INDEX CONCURRENTLY` | SUE | না | না |
| `VACUUM` | SUE | না | না |
| `VACUUM FULL` | ACCESS EXCLUSIVE | **হ্যাঁ!** | **হ্যাঁ!** |
| `ANALYZE` | SUE | না | না |
| `ALTER TABLE ADD COLUMN` (nullable, no default) | ACCESS EXCLUSIVE | **হ্যাঁ!** | **হ্যাঁ!** |
| `ALTER TABLE ADD COLUMN DEFAULT` (PG 11+) | ACCESS EXCLUSIVE | **হ্যাঁ!** | **হ্যাঁ!** |
| `ALTER TABLE SET DEFAULT` | ACCESS EXCLUSIVE | **হ্যাঁ!** | **হ্যাঁ!** |
| `ALTER TABLE ADD CONSTRAINT (NOT VALID)` | SUE | না | না |
| `ALTER TABLE VALIDATE CONSTRAINT` | SUE | না | না |
| `ALTER TABLE DROP COLUMN` | ACCESS EXCLUSIVE | **হ্যাঁ!** | **হ্যাঁ!** |
| `ALTER TABLE RENAME COLUMN` | ACCESS EXCLUSIVE | **হ্যাঁ!** | **হ্যাঁ!** |
| `TRUNCATE` | ACCESS EXCLUSIVE | **হ্যাঁ!** | **হ্যাঁ!** |
| `DROP TABLE` | ACCESS EXCLUSIVE | **হ্যাঁ!** | **হ্যাঁ!** |
| `REINDEX` | ACCESS EXCLUSIVE | **হ্যাঁ!** | **হ্যাঁ!** |
| `REINDEX CONCURRENTLY` | SUE | না | না |
| `CLUSTER` | ACCESS EXCLUSIVE | **হ্যাঁ!** | **হ্যাঁ!** |
| `REFRESH MATERIALIZED VIEW` | ACCESS EXCLUSIVE | **হ্যাঁ!** | **হ্যাঁ!** |
| `REFRESH MATERIALIZED VIEW CONCURRENTLY` | SUE | না | না |

---

## 17.4 Zero-Downtime DDL — Production Safe Patterns

### Column যোগ করা

```sql
-- ❌ Naive (blocks everything during table rewrite if large table)
ALTER TABLE orders ADD COLUMN notes TEXT DEFAULT '';

-- ✅ Safe for PostgreSQL 11+ (instant, no rewrite)
ALTER TABLE orders ADD COLUMN notes TEXT;
-- Nullable column, no default → instant, no lock beyond brief AE

-- ✅ Default সহ (PG 11+) — volatile function ছাড়া instant
ALTER TABLE orders ADD COLUMN created_by TEXT NOT NULL DEFAULT 'system';
-- PG 11+ এ default value table metadata এ store হয়, rewrite লাগে না

-- ❌ এটা slow (function call → rewrite লাগে)
ALTER TABLE orders ADD COLUMN tag TEXT DEFAULT gen_random_uuid()::text;
-- Workaround:
ALTER TABLE orders ADD COLUMN tag TEXT;                -- instant
UPDATE orders SET tag = gen_random_uuid()::text        -- batch update করো
    WHERE tag IS NULL;
ALTER TABLE orders ALTER COLUMN tag SET NOT NULL;      -- তারপর constraint
```

### Index তৈরি করা

```sql
-- ❌ DML block করে (বড় table এ minutes ধরে!)
CREATE INDEX idx_orders_status ON orders(status);

-- ✅ CONCURRENTLY (no DML block, শুধু longer time)
CREATE INDEX CONCURRENTLY idx_orders_status ON orders(status);
-- Duration: 2x longer, কিন্তু production চলতে থাকে

-- Index তৈরিতে error হলে INVALID index থাকে:
SELECT schemaname, tablename, indexname, indexdef
FROM pg_indexes
WHERE tablename = 'orders';
-- idx_orders_status INVALID থাকলে drop করে আবার করো:
DROP INDEX CONCURRENTLY idx_orders_status;
CREATE INDEX CONCURRENTLY idx_orders_status ON orders(status);
```

### Constraint যোগ করা

```sql
-- ❌ Naive — full table scan + AE lock
ALTER TABLE orders ADD CONSTRAINT fk_user
    FOREIGN KEY (user_id) REFERENCES users(id);

-- ✅ Zero-downtime approach
-- Step 1: NOT VALID দিয়ে যোগ করো (existing data check করে না, SUE lock)
ALTER TABLE orders ADD CONSTRAINT fk_user
    FOREIGN KEY (user_id) REFERENCES users(id)
    NOT VALID;

-- Step 2: Background এ validate করো (SUE lock, DML চলে)
ALTER TABLE orders VALIDATE CONSTRAINT fk_user;
-- এখন constraint fully active

-- NOT NULL constraint যোগ করা (PG 15+ fast path)
-- PG 14 এবং আগে:
ALTER TABLE orders ADD CONSTRAINT chk_status_not_null
    CHECK (status IS NOT NULL) NOT VALID;
ALTER TABLE orders VALIDATE CONSTRAINT chk_status_not_null;
-- তারপর:
ALTER TABLE orders ALTER COLUMN status SET NOT NULL;
-- PG 15+: CHECK (col IS NOT NULL) pattern দেখলে optimize করে
```

### Column Type Change

```sql
-- ❌ Full table rewrite (বড় table এ very slow + AE lock)
ALTER TABLE orders ALTER COLUMN total TYPE NUMERIC(15,4);

-- ✅ Approach: নতুন column যোগ করো, migrate করো, swap করো
-- Step 1: নতুন column যোগ করো
ALTER TABLE orders ADD COLUMN total_new NUMERIC(15,4);

-- Step 2: Trigger দিয়ে sync রাখো
CREATE OR REPLACE FUNCTION sync_total_new()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    NEW.total_new := NEW.total::NUMERIC(15,4);
    RETURN NEW;
END;
$$;
CREATE TRIGGER trg_sync_total
    BEFORE INSERT OR UPDATE ON orders
    FOR EACH ROW EXECUTE FUNCTION sync_total_new();

-- Step 3: Backfill করো (batch এ)
DO $$
DECLARE batch_size INT := 10000; updated INT;
BEGIN
  LOOP
    UPDATE orders SET total_new = total::NUMERIC(15,4)
    WHERE total_new IS NULL LIMIT batch_size;
    GET DIAGNOSTICS updated = ROW_COUNT;
    EXIT WHEN updated = 0;
    PERFORM pg_sleep(0.1);
  END LOOP;
END $$;

-- Step 4: Brief lock window এ swap (maintenance window)
BEGIN;
ALTER TABLE orders DROP TRIGGER trg_sync_total ON orders;
ALTER TABLE orders DROP COLUMN total;
ALTER TABLE orders RENAME COLUMN total_new TO total;
COMMIT;
```

---

## 17.5 Lock Monitoring এবং Kill

```sql
-- ─── সব current locks দেখো ───
SELECT
    pid,
    locktype,
    relation::regclass       AS table_name,
    mode,
    granted,
    now() - xact_start       AS txn_age,
    LEFT(query, 80)          AS query
FROM pg_locks l
JOIN pg_stat_activity a USING (pid)
WHERE relation IS NOT NULL
ORDER BY granted, txn_age DESC NULLS LAST;

-- ─── Lock wait chain দেখো (কে কাকে block করছে) ───
WITH RECURSIVE lock_chain AS (
    -- Blocked queries খোঁজো
    SELECT
        blocked.pid                     AS blocked_pid,
        blocked.usename                 AS blocked_user,
        blocked.query                   AS blocked_query,
        blocking.pid                    AS blocking_pid,
        blocking.usename                AS blocking_user,
        blocking.query                  AS blocking_query,
        now() - blocked.xact_start     AS blocked_duration,
        blocking.pid                    AS root_blocker
    FROM pg_stat_activity blocked
    JOIN pg_stat_activity blocking
        ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))

    UNION ALL

    -- Recursive: blocker কে block করছে?
    SELECT
        lc.blocked_pid,
        lc.blocked_user,
        lc.blocked_query,
        blocking.pid,
        blocking.usename,
        blocking.query,
        lc.blocked_duration,
        blocking.pid
    FROM lock_chain lc
    JOIN pg_stat_activity blocking
        ON blocking.pid = ANY(pg_blocking_pids(lc.blocking_pid))
)
SELECT DISTINCT
    blocked_pid, LEFT(blocked_query, 60) AS blocked,
    blocking_pid, LEFT(blocking_query, 60) AS blocker,
    blocked_duration
FROM lock_chain
ORDER BY blocked_duration DESC;

-- ─── application_name দিয়ে track করো ───
-- Application এ connection তৈরির সময় set করো:
-- connection string: "...&application_name=payment-service"
-- অথবা: SET application_name = 'order-processor';
SELECT application_name, count(*) FROM pg_stat_activity GROUP BY 1 ORDER BY 2 DESC;

-- ─── Kill করো ───
-- Query cancel (connection রাখে, gentle)
SELECT pg_cancel_backend(pid);
-- Connection terminate (connection close করে, forceful)
SELECT pg_terminate_backend(pid);

-- একসাথে অনেক kill করো (5 মিনিটের বেশি idle in transaction)
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND now() - xact_start > INTERVAL '5 minutes'
  AND pid != pg_backend_pid();

-- Lock এর জন্য অপেক্ষারত সব queries (3+ মিনিট)
SELECT pid, now() - xact_start AS wait_time, LEFT(query, 80)
FROM pg_stat_activity
WHERE wait_event_type = 'Lock'
  AND now() - xact_start > INTERVAL '3 minutes';
```

---

## 17.6 Lock Timeout এবং Prevention

```sql
-- ─── Lock timeout set করো ───
-- Global (postgresql.conf)
ALTER SYSTEM SET lock_timeout = '5s';
SELECT pg_reload_conf();

-- Session level (DDL এর আগে)
SET lock_timeout = '3s';
ALTER TABLE orders ADD COLUMN notes TEXT;
-- 3s এর মধ্যে lock না পেলে: ERROR: canceling statement due to lock timeout
RESET lock_timeout;

-- Transaction level
BEGIN;
SET LOCAL lock_timeout = '2s';
ALTER TABLE users ADD COLUMN phone TEXT;
COMMIT;

-- ─── Deadlock prevention ───
-- সবসময় consistent order এ lock নাও
-- BAD: Transaction A locks table1 then table2
--      Transaction B locks table2 then table1 → Deadlock!
-- GOOD: Both lock table1 then table2 → No deadlock

-- Row lock consistent order (ID ascending)
BEGIN;
SELECT * FROM accounts WHERE id IN (1, 5) ORDER BY id FOR UPDATE;
-- id=1 আগে lock, তারপর id=5
-- অন্য transaction ও একই order follow করলে deadlock হবে না
COMMIT;

-- ─── Advisory lock দিয়ে application-level mutex ───
-- একটাই process চলবে একসাথে (cron job, batch process)
BEGIN;
IF pg_try_advisory_xact_lock(12345) THEN
    -- Critical section
    PERFORM do_critical_work();
    -- Lock transaction শেষে auto-release
ELSE
    RAISE NOTICE 'Already running, skipping';
END IF;
COMMIT;
```

---

## 17.7 Production ALTER TABLE Checklist

```bash
# Production এ DDL করার আগে এই checklist follow করো:

echo "=== Pre-DDL Checklist ==="

# 1. Active connections দেখো
psql -U postgres -c "
SELECT count(*), state FROM pg_stat_activity
WHERE datname = 'mydb' GROUP BY state;"

# 2. Long running transactions আছে কিনা
psql -U postgres -c "
SELECT pid, now()-xact_start AS age, LEFT(query,80)
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
  AND now()-xact_start > INTERVAL '1 minute'
  AND datname = 'mydb'
ORDER BY age DESC;"
# থাকলে শেষ হওয়ার অপেক্ষা করো বা kill করো

# 3. Lock timeout set করো (DDL hang করলে auto-cancel)
psql -U postgres -d mydb -c "SET lock_timeout = '10s';"

# 4. Statement timeout set করো
psql -U postgres -d mydb -c "SET statement_timeout = '60s';"

# 5. DDL চালাও
psql -U postgres -d mydb -c "
SET lock_timeout = '10s';
SET statement_timeout = '60s';
ALTER TABLE orders ADD COLUMN notes TEXT;"

# 6. Verify
psql -U postgres -d mydb -c "\d orders" | grep notes
```


---

# 18. Sequence Management

## 18.1 Sequence কী এবং কীভাবে কাজ করে

```sql
-- Sequence = auto-incrementing counter
-- SERIAL/BIGSERIAL এর পেছনে sequence থাকে
-- IDENTITY column এও sequence থাকে

-- Sequence তৈরি হলে কী হয়:
CREATE TABLE users (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY
);
-- পেছনে তৈরি হয়: CREATE SEQUENCE users_id_seq
--                  START 1 INCREMENT 1 NO CYCLE

-- অথবা explicit:
CREATE SEQUENCE order_number_seq
    START WITH 1000
    INCREMENT BY 1
    MINVALUE 1000
    MAXVALUE 9999999999
    NO CYCLE;     -- শেষ হলে error (CYCLE = আবার শুরু থেকে)

-- Sequence functions
SELECT nextval('users_id_seq');    -- পরের value নাও (irreversible!)
SELECT currval('users_id_seq');    -- এই session এর current value
SELECT lastval();                  -- এই session এ সর্বশেষ nextval
SELECT setval('users_id_seq', 5000);          -- value set করো
SELECT setval('users_id_seq', 5000, false);   -- false = পরের call এ 5000 আসবে
                                               -- true  = পরের call এ 5001 আসবে
```

---

## 18.2 Sequence Gap কেন হয় (এবং কেন এটা Normal)

```
Gap হওয়ার কারণগুলো:

1. Transaction Rollback:
   BEGIN;
   INSERT INTO orders ... → sequence nextval হয়ে যায় (e.g., id=5)
   ROLLBACK;              → insert হলো না, কিন্তু sequence 5 ফেরত দেয় না!
   পরের insert → id=6
   → Gap: 5 missing

2. Cache:
   Sequence default cache=1 (PostgreSQL)
   cache=10 হলে: একসাথে 10টা value pre-allocate হয়
   Server restart হলে unused cached values হারায়
   → Gaps হয়

3. Multiple connections:
   Connection A: nextval=5
   Connection B: nextval=6
   A: rollback → id=5 gap

4. Bulk operations:
   INSERT ... SELECT → many nextval calls

Gap = Normal এবং Expected!
Gap থাকলে data integrity issue নয়।
"1,2,3,4,6,7" দেখলে panic করো না।

কখন Gap সমস্যা?
  Invoice number গুলো sequential হতে হবে (audit requirement) →
  Application level এ sequence track করো, DB sequence ব্যবহার করো না।
```

---

## 18.3 Sequence Exhaustion — Critical Problem

```sql
-- SERIAL = INTEGER = max 2,147,483,647
-- এর বেশি হলে: ERROR: integer out of range

-- ─── Risk check করো ───
SELECT
    schemaname,
    sequencename,
    data_type,
    last_value,
    CASE data_type
        WHEN 'integer' THEN 2147483647 - last_value
        WHEN 'bigint'  THEN 9223372036854775807 - last_value
    END AS remaining,
    CASE data_type
        WHEN 'integer' THEN ROUND(last_value * 100.0 / 2147483647, 2)
        WHEN 'bigint'  THEN ROUND(last_value * 100.0 / 9223372036854775807, 6)
    END AS used_pct
FROM pg_sequences
ORDER BY used_pct DESC NULLS LAST;

-- SERIAL column খোঁজো (INTEGER, at risk)
SELECT
    t.table_name,
    c.column_name,
    c.data_type,
    pg_get_serial_sequence(t.table_name, c.column_name) AS sequence_name
FROM information_schema.tables t
JOIN information_schema.columns c ON t.table_name = c.table_name
WHERE t.table_schema = 'public'
  AND c.data_type = 'integer'
  AND pg_get_serial_sequence(t.table_name, c.column_name) IS NOT NULL;
```

```sql
-- ─── Fix: INTEGER → BIGINT upgrade ───
-- Zero-downtime approach

-- Step 1: Column এবং sequence একসাথে migrate করো
-- (orders table এর id column)

-- New sequence তৈরি করো (BIGINT range)
CREATE SEQUENCE orders_id_new_seq AS BIGINT
    START WITH 1
    INCREMENT BY 1;

-- Current max value থেকে শুরু করাও
SELECT setval('orders_id_new_seq',
    (SELECT MAX(id) FROM orders) + 1000  -- safety buffer
);

-- Column type change করো (brief AE lock)
BEGIN;
-- Type change
ALTER TABLE orders ALTER COLUMN id TYPE BIGINT;
-- New sequence attach করো
ALTER TABLE orders ALTER COLUMN id
    SET DEFAULT nextval('orders_id_new_seq');
-- Old sequence ownership transfer
ALTER SEQUENCE orders_id_new_seq OWNED BY orders.id;
COMMIT;

-- Verify
SELECT pg_typeof(id) FROM orders LIMIT 1;  -- bigint
SELECT nextval('orders_id_new_seq');        -- works

-- ─── Alternative: IDENTITY column এ convert করো (PG 10+) ───
ALTER TABLE orders ALTER COLUMN id
    ADD GENERATED ALWAYS AS IDENTITY (START WITH 1000000);
-- Sequence automatically manage হবে
```

---

## 18.4 Migration পরে Sequence Reset

```sql
-- ─── Problem: pg_dump restore পরে sequence reset হয় না ───
-- Data restore হলে sequence last_value = 1 থেকে শুরু হয়
-- কিন্তু table এ already id=1..1000000 আছে
-- পরের INSERT → duplicate key error!

-- ─── Fix 1: Manual reset (একটা table) ───
SELECT setval('users_id_seq', (SELECT MAX(id) FROM users));
-- এখন nextval = MAX(id) + 1

-- ─── Fix 2: সব sequences একসাথে reset ───
-- এই query সব sequences তাদের table এর max value এ reset করে
DO $$
DECLARE
    seq_rec RECORD;
    max_val BIGINT;
BEGIN
    FOR seq_rec IN
        SELECT
            t.table_schema,
            t.table_name,
            c.column_name,
            pg_get_serial_sequence(
                t.table_schema || '.' || t.table_name,
                c.column_name
            ) AS seq_name
        FROM information_schema.tables t
        JOIN information_schema.columns c
            ON t.table_name = c.table_name
            AND t.table_schema = c.table_schema
        WHERE t.table_schema NOT IN ('information_schema', 'pg_catalog')
          AND pg_get_serial_sequence(
                t.table_schema || '.' || t.table_name,
                c.column_name) IS NOT NULL
    LOOP
        EXECUTE format(
            'SELECT COALESCE(MAX(%I), 0) FROM %I.%I',
            seq_rec.column_name,
            seq_rec.table_schema,
            seq_rec.table_name
        ) INTO max_val;

        PERFORM setval(seq_rec.seq_name, GREATEST(max_val, 1));

        RAISE NOTICE 'Reset %: → %', seq_rec.seq_name, max_val;
    END LOOP;
END $$;
```

```bash
# ─── Hands-on: Restore পরে sequence fix ───
# Step 1: Restore করো
pg_restore -U postgres -d mydb_new /backup/mydb.dump

# Step 2: Sequence check করো
psql -U postgres -d mydb_new -c "
SELECT sequencename, last_value FROM pg_sequences
WHERE schemaname = 'public'
ORDER BY sequencename;"

# Step 3: Fix script চালাও
psql -U postgres -d mydb_new -f /tmp/fix_sequences.sql

# Step 4: Verify করো
psql -U postgres -d mydb_new -c "
INSERT INTO users (username, email)
VALUES ('test_user', 'test@example.com')
RETURNING id;"
# High id দেখাবে (MAX + 1), duplicate error নয়
```

---

## 18.5 Sequence Best Practices

```sql
-- ✅ সবসময় BIGINT IDENTITY ব্যবহার করো (নতুন table এ)
id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY
-- না: id SERIAL PRIMARY KEY  (INTEGER, overflow risk)
-- না: id BIGSERIAL PRIMARY KEY  (BIGINT কিন্তু older syntax)

-- ✅ Multi-tenant এ per-tenant sequence
CREATE SEQUENCE tenant_42_order_seq START WITH 1;
SELECT nextval('tenant_42_order_seq');

-- ✅ Gap-free sequence দরকার হলে (invoice number)
-- Advisory lock দিয়ে:
CREATE TABLE invoice_numbers (
    id          SERIAL PRIMARY KEY,
    year        INTEGER,
    last_number INTEGER DEFAULT 0,
    UNIQUE (year)
);

CREATE OR REPLACE FUNCTION next_invoice_number(p_year INTEGER)
RETURNS TEXT
LANGUAGE plpgsql
AS $$
DECLARE
    v_num INTEGER;
BEGIN
    -- Advisory lock নাও (gap-free guarantee)
    PERFORM pg_advisory_xact_lock(hashtext('invoice_number_' || p_year));

    INSERT INTO invoice_numbers (year, last_number)
    VALUES (p_year, 1)
    ON CONFLICT (year) DO UPDATE
        SET last_number = invoice_numbers.last_number + 1
    RETURNING last_number INTO v_num;

    RETURN p_year || '-' || LPAD(v_num::TEXT, 6, '0');
END;
$$;

SELECT next_invoice_number(2024);  -- 2024-000001
SELECT next_invoice_number(2024);  -- 2024-000002
-- Gap হবে না (advisory lock guarantee করে)
```


---

# 19. JSONB Advanced Patterns

## 19.1 JSON Path — PG 12+

```sql
-- jsonb_path_query: SQL/JSON path language
-- JSON এর মধ্যে deep navigate করো

SELECT jsonb_path_query(
    '{"store": {"books": [{"title": "PG Guide", "price": 29.99},
                           {"title": "SQL Deep Dive", "price": 39.99}]}}',
    '$.store.books[*].title'
);
-- "PG Guide"
-- "SQL Deep Dive"

-- Filter করো
SELECT jsonb_path_query(
    '{"store": {"books": [{"title": "PG Guide", "price": 29.99},
                           {"title": "SQL", "price": 9.99}]}}',
    '$.store.books[*] ? (@.price > 20)'
);
-- {"title": "PG Guide", "price": 29.99}

-- Table এ ব্যবহার করো
SELECT jsonb_path_query(metadata, '$.tags[*]') AS tag
FROM products
WHERE jsonb_path_exists(metadata, '$.price ? (@ > 100)');

-- jsonb_path_match: boolean result
SELECT * FROM products
WHERE jsonb_path_match(metadata, '$.in_stock == true');
```

---

## 19.2 JSONB Schema Validation এবং Transformation

```sql
-- ─── jsonb_populate_record: JSON → Row ───
CREATE TYPE product_type AS (
    name TEXT, price NUMERIC, category TEXT
);

SELECT * FROM jsonb_populate_record(
    NULL::product_type,
    '{"name": "Laptop", "price": 999.99, "category": "electronics"}'
);
-- name   | price  | category
-- Laptop | 999.99 | electronics

-- Multiple rows থেকে
SELECT p.* FROM
    jsonb_populate_recordset(NULL::product_type,
        '[{"name": "Laptop", "price": 999.99},
          {"name": "Mouse",  "price": 29.99}]'
    ) p;

-- ─── JSONB Aggregation ───
-- rows → JSONB object
SELECT jsonb_object_agg(username, email) FROM users LIMIT 10;
-- {"alice": "alice@...", "bob": "bob@..."}

-- rows → JSONB array of objects
SELECT jsonb_agg(jsonb_build_object(
    'id', id,
    'name', username,
    'created', created_at
) ORDER BY id) AS users_json
FROM users
WHERE status = 'active';

-- ─── JSONB Update (immutable এর পরিবর্তে) ───
-- একটা key update করো
UPDATE products
SET metadata = jsonb_set(metadata, '{price}', '149.99')
WHERE id = 5;

-- Nested update
UPDATE products
SET metadata = jsonb_set(metadata, '{specs, ram}', '"32GB"')
WHERE id = 5;

-- Key delete করো
UPDATE products
SET metadata = metadata - 'discount_code'
WHERE id = 5;

-- Array element delete (index 0)
UPDATE products
SET metadata = metadata #- '{tags, 0}'
WHERE id = 5;

-- Multiple key update
UPDATE products
SET metadata = metadata
    || jsonb_build_object('updated_at', NOW()::text, 'version', 2)
WHERE id = 5;

-- ─── JSONB Validation (custom function) ───
CREATE OR REPLACE FUNCTION validate_product_metadata(data JSONB)
RETURNS BOOLEAN
LANGUAGE plpgsql IMMUTABLE
AS $$
BEGIN
    -- Required fields check
    IF NOT (data ? 'name' AND data ? 'price') THEN
        RETURN FALSE;
    END IF;
    -- Type check
    IF jsonb_typeof(data -> 'price') != 'number' THEN
        RETURN FALSE;
    END IF;
    -- Range check
    IF (data ->> 'price')::NUMERIC < 0 THEN
        RETURN FALSE;
    END IF;
    RETURN TRUE;
END;
$$;

ALTER TABLE products ADD CONSTRAINT chk_metadata
    CHECK (validate_product_metadata(metadata));
```

---

# 20. Full-Text Search — Deep Dive

## 20.1 tsvector এবং tsquery — কীভাবে কাজ করে

```sql
-- tsvector = processed, searchable form of text
-- tsquery  = search query

-- tsvector তৈরি করো
SELECT to_tsvector('english', 'The quick brown fox jumps over the lazy dog');
-- 'brown':3 'dog':9 'fox':4 'jump':5 'lazi':8 'quick':2
-- ↑ Stop words (the, over) removed
-- ↑ Words stemmed (jumps→jump, lazy→lazi)
-- ↑ Position numbers (search ranking এ কাজ লাগে)

-- tsquery তৈরি করো
SELECT to_tsquery('english', 'quick & fox');      -- AND
SELECT to_tsquery('english', 'quick | slow');     -- OR
SELECT to_tsquery('english', '!slow');            -- NOT
SELECT to_tsquery('english', 'quick <-> fox');    -- FOLLOWED BY (adjacent)
SELECT to_tsquery('english', 'quic:*');           -- prefix match

-- plainto_tsquery: plain text → tsquery (user input এর জন্য)
SELECT plainto_tsquery('english', 'quick brown fox');  -- 'quick' & 'brown' & 'fox'

-- phraseto_tsquery: phrase search
SELECT phraseto_tsquery('english', 'quick brown');  -- phrase এ order matter করে

-- websearch_to_tsquery: Google-style syntax
SELECT websearch_to_tsquery('english', 'quick fox -slow "brown dog"');
```

---

## 20.2 Full-Text Search Implementation

```sql
-- ─── Setup ───
-- Generated column দিয়ে (automatic update, PG 12+)
ALTER TABLE articles
    ADD COLUMN search_vector TSVECTOR
    GENERATED ALWAYS AS (
        setweight(to_tsvector('english', COALESCE(title, '')), 'A') ||
        setweight(to_tsvector('english', COALESCE(summary, '')), 'B') ||
        setweight(to_tsvector('english', COALESCE(body, '')), 'C')
    ) STORED;
-- setweight: A=most important, B, C, D=least
-- Title match → higher rank than body match

-- GIN Index
CREATE INDEX idx_articles_fts ON articles USING GIN(search_vector);

-- ─── Search করো ───
SELECT
    id,
    title,
    ts_rank(search_vector, query)        AS rank,
    ts_headline('english', body, query,  -- highlighted snippet
        'MaxWords=30, MinWords=10, StartSel=<mark>, StopSel=</mark>'
    ) AS snippet
FROM articles,
    to_tsquery('english', 'postgresql & performance') AS query
WHERE search_vector @@ query
ORDER BY rank DESC
LIMIT 10;

-- ─── Ranking ───
-- ts_rank: word frequency based
-- ts_rank_cd: cover density (phrase closeness)
SELECT title,
    ts_rank(search_vector, query)    AS freq_rank,
    ts_rank_cd(search_vector, query) AS cd_rank
FROM articles,
    to_tsquery('english', 'database & performance') query
WHERE search_vector @@ query
ORDER BY cd_rank DESC;
```

---

## 20.3 Multilingual Full-Text Search

```sql
-- Available configurations দেখো
SELECT cfgname FROM pg_ts_config;
-- english, simple, arabic, french, german, etc.

-- 'simple' = no stemming, no stop words (good for names/IDs)
SELECT to_tsvector('simple', 'PostgreSQL DBA Guide');
-- 'dba':2 'guide':3 'postgresql':1

-- Multiple language support
CREATE TABLE multilang_articles (
    id       BIGINT PRIMARY KEY,
    lang     TEXT,  -- 'english', 'german', etc.
    title    TEXT,
    body     TEXT
);

-- Language-specific search
SELECT * FROM multilang_articles
WHERE to_tsvector(lang::regconfig, title || ' ' || body)
    @@ to_tsquery(lang::regconfig, 'database');

-- ─── unaccent extension (accent-insensitive) ───
CREATE EXTENSION unaccent;

-- unaccent dictionary তৈরি করো
CREATE TEXT SEARCH CONFIGURATION english_unaccent (COPY = english);
ALTER TEXT SEARCH CONFIGURATION english_unaccent
    ALTER MAPPING FOR hword, hword_part, word
    WITH unaccent, english_stem;

-- এখন "café" এবং "cafe" দুটোই match করবে
SELECT to_tsvector('english_unaccent', 'café résumé naïve');
-- 'cafe':1 'naiv':3 'resum':2
```

---

## 20.4 pg_trgm দিয়ে Fuzzy Search

```sql
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_trgm ON products USING GIN(name gin_trgm_ops);

-- Fuzzy match (typo tolerant)
SELECT name, similarity(name, 'postgresq') AS score
FROM products
WHERE similarity(name, 'postgresq') > 0.3
ORDER BY score DESC;

-- LIKE এ index use
SELECT * FROM products WHERE name ILIKE '%postgre%';  -- index use করবে

-- Combined: FTS + trigram
SELECT p.name, ts_rank(p.search_vector, q.query) AS rank
FROM products p,
    plainto_tsquery('english', 'postgres database') q
WHERE p.search_vector @@ q.query
   OR similarity(p.name, 'postgres') > 0.4
ORDER BY rank DESC, similarity(p.name, 'postgres') DESC;
```

---

# 21. Foreign Data Wrappers (FDW)

## 21.1 postgres_fdw — PostgreSQL থেকে PostgreSQL Query

```sql
-- ─── Setup ───
CREATE EXTENSION postgres_fdw;

-- Remote server define করো
CREATE SERVER remote_pg
    FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (host '172.16.93.145', port '5432', dbname 'analytics_db');

-- User mapping (local user → remote user)
CREATE USER MAPPING FOR postgres
    SERVER remote_pg
    OPTIONS (user 'readonly_user', password 'RemotePass@2024!');

-- Remote table import করো
IMPORT FOREIGN SCHEMA public
    LIMIT TO (orders, products)
    FROM SERVER remote_pg
    INTO remote_analytics;  -- local schema

-- অথবা specific table:
CREATE FOREIGN TABLE remote_orders (
    id          BIGINT,
    user_id     INTEGER,
    total       NUMERIC,
    created_at  TIMESTAMPTZ
)
SERVER remote_pg
OPTIONS (schema_name 'public', table_name 'orders');
```

```sql
-- ─── Query করো ───
-- Local এর মতোই query করো
SELECT * FROM remote_analytics.orders WHERE user_id = 5;

-- Local এবং Remote join করো
SELECT u.name, r.total, r.created_at
FROM users u          -- local table
JOIN remote_analytics.orders r ON u.id = r.user_id  -- remote table
WHERE r.created_at > NOW() - INTERVAL '30 days';

-- ─── Performance considerations ───
-- FDW pushdown: filter, join, aggregate remote এ চলতে পারে
EXPLAIN SELECT COUNT(*) FROM remote_analytics.orders WHERE user_id = 5;
-- "Foreign Scan" with "Filter" pushed down দেখাবে ভালো

-- FDW options tuning
ALTER SERVER remote_pg OPTIONS (
    ADD use_remote_estimate 'true',  -- remote statistics use করো
    ADD fdw_startup_cost '100',
    ADD fdw_tuple_cost '0.2'
);

-- ─── Write করো (INSERT/UPDATE/DELETE) ───
INSERT INTO remote_analytics.orders (user_id, total)
VALUES (5, 100.00);  -- remote table এ insert হবে

UPDATE remote_analytics.orders SET total = 200
WHERE id = 1;
```

---

## 21.2 file_fdw — CSV/File থেকে Read

```sql
CREATE EXTENSION file_fdw;

CREATE SERVER file_server FOREIGN DATA WRAPPER file_fdw;

CREATE FOREIGN TABLE csv_logs (
    log_time    TEXT,
    level       TEXT,
    message     TEXT
)
SERVER file_server
OPTIONS (
    filename '/var/log/app/access.log',
    format 'csv',
    header 'true'
);

-- Query log file directly!
SELECT level, COUNT(*) FROM csv_logs GROUP BY level;
SELECT * FROM csv_logs WHERE level = 'ERROR' LIMIT 10;
```

---

## 21.3 mysql_fdw — MySQL থেকে Read

```bash
# Install
sudo dnf install -y mysql_fdw_16
```

```sql
CREATE EXTENSION mysql_fdw;

CREATE SERVER mysql_server
    FOREIGN DATA WRAPPER mysql_fdw
    OPTIONS (host '172.16.93.146', port '3306');

CREATE USER MAPPING FOR postgres
    SERVER mysql_server
    OPTIONS (username 'root', password 'MySQLPass@2024!');

IMPORT FOREIGN SCHEMA mydb
    FROM SERVER mysql_server
    INTO mysql_data;

-- MySQL table query করো PostgreSQL থেকে!
SELECT * FROM mysql_data.users LIMIT 10;

-- Migration তে useful:
INSERT INTO local_users
SELECT id, username, email, created_at
FROM mysql_data.users;
```

---

# 22. Extended Statistics এবং Advanced Planner Tuning

## 22.1 CREATE STATISTICS — Multi-Column Correlation

```sql
-- Problem: Planner estimate ভুল হয় যখন columns correlated
-- Example: city='Dhaka' AND country='BD' → planner thinks independent
--          কিন্তু actually city এবং country strongly correlated

-- Check করো estimate কতটা off
EXPLAIN ANALYZE
SELECT * FROM users WHERE city = 'Dhaka' AND country = 'BD';
-- rows=X (actual rows=Y) — X এবং Y অনেক আলাদা হলে problem

-- ─── Extended Statistics তৈরি করো ───
-- MCV (Most Common Values) statistics
CREATE STATISTICS stat_city_country (mcv)
    ON city, country FROM users;

-- Functional dependencies
CREATE STATISTICS stat_zip_city (dependencies)
    ON zip_code, city, state FROM addresses;
-- zip_code জানলে city/state জানা যায় → planner এই correlation বুঝবে

-- N-distinct combinations
CREATE STATISTICS stat_user_status (ndistinct)
    ON user_id, status FROM orders;

-- সব types একসাথে
CREATE STATISTICS stat_order_details
    ON user_id, status, created_at FROM orders;

-- Statistics update করো
ANALYZE users;
ANALYZE addresses;

-- ─── Verify improvement ───
EXPLAIN ANALYZE
SELECT * FROM users WHERE city = 'Dhaka' AND country = 'BD';
-- rows=X (actual rows=Y) — এখন অনেক কাছাকাছি হওয়া উচিত

-- Statistics দেখো
SELECT stxname, stxkind, stxrelid::regclass AS table_name
FROM pg_statistic_ext;
```

---

## 22.2 Connection Management — Complete Picture

### pgBouncer SCRAM-SHA-256 Setup

```bash
# ─── Step 1: PostgreSQL তে SCRAM auth confirm করো ───
psql -U postgres -c "SHOW password_encryption;"
# scram-sha-256 হওয়া উচিত

# ─── Step 2: Application user তৈরি করো ───
psql -U postgres -c "
CREATE USER appuser PASSWORD 'AppPass@2024!';
GRANT CONNECT ON DATABASE mydb TO appuser;
GRANT USAGE ON SCHEMA public TO appuser;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO appuser;"

# ─── Step 3: pgBouncer userlist.txt তৈরি করো ───
# SCRAM hash PostgreSQL থেকে নাও:
psql -U postgres -t -A -c "
SELECT '\"' || usename || '\" \"' || passwd || '\"'
FROM pg_shadow
WHERE usename IN ('appuser', 'postgres');" > /etc/pgbouncer/userlist.txt

cat /etc/pgbouncer/userlist.txt
# "appuser" "SCRAM-SHA-256$4096:..."
# "postgres" "SCRAM-SHA-256$4096:..."

# ─── Step 4: pgbouncer.ini configure করো ───
sudo nano /etc/pgbouncer/pgbouncer.ini
```

```ini
[databases]
mydb = host=127.0.0.1 port=5432 dbname=mydb

[pgbouncer]
listen_addr         = *
listen_port         = 6432
auth_type           = scram-sha-256   # ← SCRAM
auth_file           = /etc/pgbouncer/userlist.txt
pool_mode           = transaction
default_pool_size   = 25
max_client_conn     = 500
reserve_pool_size   = 5
server_idle_timeout = 600
log_connections     = 1
log_disconnections  = 1
admin_users         = postgres
```

```bash
# ─── Step 5: pg_hba.conf এ pgBouncer allow করো ───
# pgBouncer → PostgreSQL connection
# (pgBouncer local এ চললে localhost দিয়ে connect)
# host mydb appuser 127.0.0.1/32 scram-sha-256

sudo systemctl restart pgbouncer

# ─── Step 6: Verify ───
psql -h localhost -p 6432 -U appuser -d mydb -c "SELECT current_user, inet_server_addr();"
```

### Pool Size Formula

```
Pool size কতটা দিতে হবে?

Formula:
  default_pool_size = (DB CPU cores × 2) + effective_spindle_count
  
  Example: 4 CPU, SSD (1 spindle effectively):
  pool_size = (4 × 2) + 1 = 9

  কিন্তু practical এ:
  pool_size = max_connections / (number_of_pgbouncer_instances × users)
  
  Example:
  max_connections = 100 (PostgreSQL এ)
  pgBouncer instances = 1
  Different users = 5
  pool_size per user = 100 / (1 × 5) = 20

  max_client_conn:
  = application threads × (1.1 to 1.5 safety buffer)
  Application 200 threads চালায়:
  max_client_conn = 200 × 1.2 = 240
```

### Read-Your-Own-Writes Problem

```python
# Problem:
# User writes → Primary
# User reads → Replica (may not have the write yet!)
# User sees stale data

# Solution 1: Write 후 같은 connection에서 읽기
# Application 에서 write 후 primary에서 읽기:
with db.primary_connection() as conn:
    conn.execute("INSERT INTO posts ...")
    result = conn.execute("SELECT * FROM posts WHERE ...")  # primary에서 읽기

# Solution 2: Session variable 사용
# Write 후 LSN 기록
lsn = conn.execute("SELECT pg_current_wal_lsn()").scalar()

# Read replica에서 이 LSN까지 replay됐는지 확인
replica_conn.execute(
    "SELECT pg_wal_lsn_diff(pg_last_wal_replay_lsn(), %s)",
    (lsn,)
)
# 양수면 replica가 따라잡음 → 읽어도 됨

# Solution 3: Synchronous commit for critical writes
conn.execute("SET synchronous_commit = 'remote_apply'")
conn.execute("INSERT INTO critical_table ...")
conn.execute("RESET synchronous_commit")
# 이후 모든 replica에서 읽어도 안전
```

---

## 22.3 Post-Upgrade Checklist

```bash
# ─── pg_upgrade বা Logical Replication দিয়ে upgrade পরে ───

echo "=== Post-Upgrade Checklist ==="

# 1. Version verify করো
psql -U postgres -c "SELECT version();"

# 2. Statistics rebuild (pg_upgrade statistics migrate করে না)
echo "Rebuilding statistics..."
sudo -u postgres /usr/pgsql-16/bin/vacuumdb \
    --all --analyze-in-stages -U postgres -p 5432
# Stage 1: minimal statistics → faster start
# Stage 2: default statistics
# Stage 3: full statistics

# 3. Extension compatibility check
psql -U postgres -d mydb -c "
SELECT name, default_version, installed_version,
    CASE WHEN default_version != installed_version
         THEN 'UPGRADE NEEDED'
         ELSE 'OK'
    END AS status
FROM pg_available_extensions
WHERE installed_version IS NOT NULL;"
# Upgrade needed হলে:
# ALTER EXTENSION pgcrypto UPDATE;

# 4. Invalid indexes দেখো (pg_upgrade পরে হতে পারে)
psql -U postgres -d mydb -c "
SELECT schemaname, tablename, indexname
FROM pg_indexes
WHERE indexname NOT IN (
    SELECT indexname FROM pg_stat_user_indexes
);"
# অথবা:
psql -U postgres -d mydb -c "
SELECT relname, pg_get_indexdef(i.indexrelid)
FROM pg_index i
JOIN pg_class c ON c.oid = i.indrelid
WHERE NOT i.indisvalid;"
# INVALID index → DROP CONCURRENTLY → CREATE CONCURRENTLY

# 5. Sequence ownership verify করো
psql -U postgres -d mydb -c "
SELECT sequencename, last_value FROM pg_sequences
WHERE schemaname = 'public'
ORDER BY sequencename;"

# 6. Foreign key constraints valid কিনা
psql -U postgres -d mydb -c "
SELECT conname, conrelid::regclass, confrelid::regclass
FROM pg_constraint
WHERE contype = 'f' AND NOT convalidated;"
# NOT validated থাকলে: ALTER TABLE t VALIDATE CONSTRAINT c;

# 7. Configuration comparison (old vs new)
diff /tmp/old_postgresql.conf /var/lib/pgsql/16/data/postgresql.conf
# Custom settings migrate হয়েছে কিনা দেখো

# 8. Application connection test
psql -h localhost -U appuser -d mydb -c "SELECT current_user;"

# 9. Performance baseline compare করো
pgbench -U postgres -c 10 -T 60 mydb 2>&1 | grep tps
# Old baseline এর সাথে compare করো

# 10. Replication re-establish করো (pg_upgrade এর পরে)
# Standbys কে re-clone করতে হবে!
pg_basebackup -h 172.16.93.140 -U replicator \
    -D /var/lib/pgsql/16/data -Xs -R -P

echo "Post-upgrade checklist complete!"
```


---

# 23. Advanced PL/pgSQL

## 23.1 Dynamic SQL

```sql
-- ─── EXECUTE দিয়ে Dynamic SQL ───
-- Table name, column name dynamically pass করতে হলে
-- (Regular SQL এ table/column name parameter হতে পারে না)

CREATE OR REPLACE FUNCTION get_table_count(p_table TEXT)
RETURNS BIGINT
LANGUAGE plpgsql
AS $$
DECLARE
    v_count BIGINT;
    v_sql   TEXT;
BEGIN
    -- format() দিয়ে safe query build করো (SQL injection prevent)
    v_sql := format('SELECT COUNT(*) FROM %I', p_table);
    -- %I = identifier (table/column name) — automatically quoted
    -- %L = literal (value) — safely quoted
    -- %s = string (no quoting — dangerous for user input!)

    EXECUTE v_sql INTO v_count;
    RETURN v_count;
END;
$$;

SELECT get_table_count('orders');   -- safe
SELECT get_table_count('users');    -- safe

-- ─── Dynamic column ───
CREATE OR REPLACE FUNCTION get_column_stats(
    p_table  TEXT,
    p_column TEXT
)
RETURNS TABLE(min_val TEXT, max_val TEXT, count_val BIGINT)
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY EXECUTE format(
        'SELECT MIN(%I)::TEXT, MAX(%I)::TEXT, COUNT(%I)
         FROM %I',
        p_column, p_column, p_column, p_table
    );
END;
$$;

SELECT * FROM get_column_stats('orders', 'total');

-- ─── Dynamic WHERE clause ───
CREATE OR REPLACE FUNCTION search_users(
    p_status   TEXT DEFAULT NULL,
    p_country  TEXT DEFAULT NULL,
    p_min_age  INTEGER DEFAULT NULL
)
RETURNS SETOF users
LANGUAGE plpgsql
AS $$
DECLARE
    v_sql    TEXT := 'SELECT * FROM users WHERE 1=1';
    v_params TEXT[] := '{}';
    v_idx    INTEGER := 1;
BEGIN
    IF p_status IS NOT NULL THEN
        v_sql := v_sql || format(' AND status = $%s', v_idx);
        v_params := v_params || p_status;
        v_idx := v_idx + 1;
    END IF;

    IF p_country IS NOT NULL THEN
        v_sql := v_sql || format(' AND country = $%s', v_idx);
        v_params := v_params || p_country;
        v_idx := v_idx + 1;
    END IF;

    IF p_min_age IS NOT NULL THEN
        v_sql := v_sql || format(' AND age >= $%s', v_idx);
        v_params := v_params || p_min_age::TEXT;
    END IF;

    RETURN QUERY EXECUTE v_sql USING
        v_params[1], v_params[2], v_params[3];
END;
$$;

SELECT * FROM search_users(p_status => 'active', p_country => 'BD');
```

---

## 23.2 Cursor — Large Result Set Processing

```sql
-- Cursor দরকার যখন:
-- ① লাখো rows process করতে হবে কিন্তু একসাথে memory তে রাখা যাবে না
-- ② Row-by-row processing দরকার

CREATE OR REPLACE PROCEDURE process_large_table()
LANGUAGE plpgsql
AS $$
DECLARE
    -- Cursor declare করো
    cur CURSOR FOR
        SELECT id, total FROM orders WHERE processed = false
        ORDER BY id;

    v_rec    RECORD;
    v_batch  INTEGER := 0;
    v_total  INTEGER := 0;
BEGIN
    OPEN cur;   -- Cursor open করো

    LOOP
        FETCH cur INTO v_rec;   -- পরের row নাও
        EXIT WHEN NOT FOUND;    -- আর row নেই → exit

        -- Process করো
        UPDATE orders
        SET processed = true,
            processed_at = NOW()
        WHERE id = v_rec.id;

        v_batch := v_batch + 1;
        v_total := v_total + 1;

        -- প্রতি 1000 rows এ commit করো (memory/lock management)
        IF v_batch >= 1000 THEN
            COMMIT;
            v_batch := 0;
            RAISE NOTICE 'Processed % rows total', v_total;
        END IF;
    END LOOP;

    CLOSE cur;  -- Cursor close করো
    COMMIT;     -- Final commit
    RAISE NOTICE 'Done. Total processed: %', v_total;
END;
$$;

CALL process_large_table();

-- ─── Scrollable Cursor (forward/backward) ───
DECLARE
    cur SCROLL CURSOR FOR SELECT * FROM orders ORDER BY id;
BEGIN
    OPEN cur;
    FETCH FIRST FROM cur INTO v_rec;    -- First row
    FETCH LAST FROM cur INTO v_rec;     -- Last row
    FETCH PRIOR FROM cur INTO v_rec;    -- Previous row
    FETCH ABSOLUTE 100 FROM cur INTO v_rec;  -- 100th row
    FETCH RELATIVE -5 FROM cur INTO v_rec;   -- 5 rows back
    CLOSE cur;
END;
```

---

## 23.3 LISTEN / NOTIFY — Async Messaging

```sql
-- PostgreSQL built-in pub/sub!
-- Real-time notification → application কে alert করো

-- ─── NOTIFY (Publisher) ───
-- Database change হলে notification পাঠাও
NOTIFY channel_name;                    -- simple
NOTIFY channel_name, 'payload text';   -- with payload (max 8000 bytes)

-- Trigger দিয়ে auto-notify
CREATE OR REPLACE FUNCTION fn_notify_order_change()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    -- JSON payload তৈরি করো
    PERFORM pg_notify(
        'order_changes',
        json_build_object(
            'operation', TG_OP,
            'order_id',  NEW.id,
            'status',    NEW.status,
            'timestamp', NOW()
        )::text
    );
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_notify_orders
    AFTER INSERT OR UPDATE ON orders
    FOR EACH ROW EXECUTE FUNCTION fn_notify_order_change();

-- ─── LISTEN (Subscriber) ───
LISTEN order_changes;

-- Notification আসলে:
-- Asynchronous notification "order_changes" with payload:
-- {"operation":"UPDATE","order_id":5,"status":"completed"}

-- ─── Application এ (Python) ───
-- import psycopg2, select
-- conn = psycopg2.connect(...)
-- conn.set_isolation_level(0)  # autocommit
-- cur = conn.cursor()
-- cur.execute("LISTEN order_changes")
-- while True:
--     select.select([conn], [], [], 5)  # wait 5s
--     conn.poll()
--     while conn.notifies:
--         notify = conn.notifies.pop()
--         print(f"Got: {notify.payload}")

-- ─── Use cases ───
-- Cache invalidation: DB change → notify → cache clear
-- Real-time dashboard: data update → notify → WebSocket push
-- Job queue: new job → notify → worker wakes up
-- Replication: custom change data capture (CDC)
```

---

## 23.4 GET DIAGNOSTICS — Operation Result Info

```sql
CREATE OR REPLACE PROCEDURE batch_process_orders()
LANGUAGE plpgsql
AS $$
DECLARE
    v_rows_updated    INTEGER;
    v_rows_inserted   INTEGER;
    v_context         TEXT;
BEGIN
    -- Update করো
    UPDATE orders
    SET status = 'processing'
    WHERE status = 'pending'
      AND created_at < NOW() - INTERVAL '1 hour';

    -- কতটা row affected হয়েছে জানো
    GET DIAGNOSTICS v_rows_updated = ROW_COUNT;
    RAISE NOTICE 'Updated % orders to processing', v_rows_updated;

    -- Insert করো
    INSERT INTO processing_log (batch_time, order_count)
    VALUES (NOW(), v_rows_updated);

    GET DIAGNOSTICS v_rows_inserted = ROW_COUNT;

    -- Exception handler এ call stack দেখো
EXCEPTION WHEN OTHERS THEN
    GET STACKED DIAGNOSTICS
        v_context = PG_EXCEPTION_CONTEXT;
    RAISE NOTICE 'Error context: %', v_context;
    RAISE;
END;
$$;

-- Available GET DIAGNOSTICS variables:
-- ROW_COUNT          → affected rows
-- PG_CONTEXT         → call stack
-- PG_EXCEPTION_DETAIL → error detail
-- PG_EXCEPTION_HINT   → error hint
-- RETURNED_SQLSTATE   → SQL state code
```

---

## 23.5 Row-Level Security এ Function

```sql
-- ─── Multi-tenant RLS দিয়ে ───
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders FORCE ROW LEVEL SECURITY;

-- current_setting() দিয়ে tenant context পাঠাও
CREATE POLICY tenant_isolation ON orders
    USING (tenant_id = current_setting('app.tenant_id')::integer);

-- Application: প্রতিটা request এ set করো
SET app.tenant_id = '42';
SELECT * FROM orders;  -- শুধু tenant 42 এর data

-- ─── Function দিয়ে সহজ করো ───
CREATE OR REPLACE FUNCTION set_tenant(p_tenant_id INTEGER)
RETURNS VOID
LANGUAGE plpgsql
SECURITY DEFINER   -- caller permission এ নয়, owner permission এ চলে
AS $$
BEGIN
    PERFORM set_config('app.tenant_id', p_tenant_id::text, true);
    -- true = local (transaction end এ reset)
    -- false = persistent (session শেষ পর্যন্ত)
END;
$$;

SELECT set_tenant(42);
SELECT * FROM orders;  -- tenant 42 only
```
