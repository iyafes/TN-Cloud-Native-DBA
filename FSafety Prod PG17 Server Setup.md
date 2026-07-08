# PostgreSQL 17 Installation, Configuration & Setup Guide
## Target: Rocky Linux 9 — Binary Install (No Docker)

---

## Overview

This document covers the complete process of installing PostgreSQL 17 on a fresh Rocky Linux 9 server with custom data and log directories on separate NVMe disks. It includes all issues encountered and their fixes so the process can be completed in one attempt.

**Server Specs (this setup was tested on):**
- OS: Rocky Linux 9
- RAM: 8GB
- CPU: 4 cores
- Disk layout:
  - `/` (root): 30GB NVMe (OS only)
  - `/data`: 50GB NVMe (database data)
  - `/logs`: 50GB NVMe (database logs)

---

## Pre-requisites

- A pre-existing PostgreSQL installation (e.g., PG 16) must be cleanly removed before installing PG 17.
- SELinux is **Enforcing** — do not disable it. Apply correct contexts instead (covered below).
- No swap configured by default on this server — add swap before starting (see Optional section).

---

## Phase 1 — Remove Existing PostgreSQL (PG 16)

If a previous version of PostgreSQL is installed (e.g., PG 16), remove it completely first.

### 1.1 Stop and Disable the Service

```bash
sudo systemctl stop postgresql-16
sudo systemctl disable postgresql-16
sudo systemctl status postgresql-16
# Expected: inactive (dead)
```

### 1.2 Remove Data Directory and Packages

> **Important:** Verify there is no production data in the old data directory before removing.

Check if old data directory has any meaningful data:
```bash
sudo -u postgres /usr/pgsql-16/bin/psql -c "\l+"
# If only default databases (postgres, template0, template1) exist, it is safe to remove.
```

Remove data directory and packages:
```bash
sudo rm -rf /data/pgsql/16/

sudo dnf remove -y postgresql16 postgresql16-server postgresql16-libs postgresql16-contrib 2>/dev/null
sudo rm -rf /usr/pgsql-16/
sudo dnf clean all
```

Verify clean removal:
```bash
rpm -qa | grep postgresql
# Expected: no output
```

---

## Phase 2 — Install PostgreSQL 17

### 2.1 Add PGDG Repository

Rocky Linux 9 is an EL9 system. Use the EL-9 repo URL specifically.

```bash
sudo dnf install -y \
  https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```

### 2.2 Disable Built-in PostgreSQL Module

Rocky Linux ships with a built-in PostgreSQL module that conflicts with PGDG packages. Disable it:

```bash
sudo dnf module disable postgresql -y
```

### 2.3 Install PG 17 Packages

```bash
sudo dnf install -y postgresql17 postgresql17-server postgresql17-contrib
```

Verify installation:
```bash
/usr/pgsql-17/bin/postgres --version
# Expected: postgres (PostgreSQL) 17.x
```

---

## Phase 3 — Create Custom Directories

Do **not** use default PostgreSQL paths. Data goes to `/data`, logs go to `/logs`.

```bash
# PostgreSQL data directory
sudo mkdir -p /data/pgsql/17/data
sudo chown -R postgres:postgres /data/pgsql/17/data
sudo chmod 700 /data/pgsql/17/data

# PostgreSQL log directory
sudo mkdir -p /logs/postgresql
sudo chown -R postgres:postgres /logs/postgresql
sudo chmod 700 /logs/postgresql

# WAL archive directory (required for backup)
sudo mkdir -p /logs/archive
sudo chown -R postgres:postgres /logs/archive
sudo chmod 700 /logs/archive
```

Verify:
```bash
ls -la /data/pgsql/17/
ls -la /logs/
```

---

## Phase 4 — SELinux Context (Critical on Rocky Linux)

SELinux is Enforcing on Rocky Linux. Without proper context labels, PostgreSQL cannot write to custom directories. **Do not disable SELinux.**

The parent directories `/data` and `/logs` are on separate NVMe disks and start with `unlabeled_t` context. Fix parent directories first, then apply specific contexts.

### 4.1 Fix Parent Directory Contexts

```bash
sudo semanage fcontext -a -t var_t "/data(/.*)?"
sudo restorecon -Rv /data

sudo semanage fcontext -a -t var_log_t "/logs(/.*)?"
sudo restorecon -Rv /logs
```

### 4.2 Apply PostgreSQL-Specific Contexts

```bash
sudo semanage fcontext -a -t postgresql_db_t "/data/pgsql(/.*)?"
sudo semanage fcontext -a -t postgresql_log_t "/logs/postgresql(/.*)?"
sudo semanage fcontext -a -t postgresql_log_t "/logs/archive(/.*)?"

sudo chcon -Rv -t postgresql_db_t /data/pgsql
sudo chcon -Rv -t postgresql_log_t /logs/postgresql
sudo chcon -Rv -t postgresql_log_t /logs/archive
```

> **Note:** `chcon` applies context immediately. The `semanage fcontext` entries make it persistent across reboots. Both are needed.

Verify:
```bash
ls -laZ /data/pgsql/17/data/
ls -laZ /logs/postgresql/
ls -laZ /logs/archive/
# Expected: postgresql_db_t and postgresql_log_t contexts respectively
```

---

## Phase 5 — Initialize the Database Cluster (initdb)

```bash
sudo -u postgres /usr/pgsql-17/bin/initdb \
  -D /data/pgsql/17/data \
  --encoding=UTF8 \
  --locale=en_US.UTF-8 \
  --data-checksums
```

`--data-checksums` enables page-level checksums to detect disk corruption — essential for production.

Expected output ends with:
```
Success. You can now start the database server using: ...
```

Verify files were created:
```bash
sudo ls -la /data/pgsql/17/data/
# Expected: pg_hba.conf, postgresql.conf, pg_wal/, base/, global/ etc.
```

---

## Phase 6 — Configure systemd Service (Custom PGDATA)

The default service file points to a different PGDATA. Update it to use the custom data directory.

```bash
sudo sed -i 's|Environment=PGDATA=.*|Environment=PGDATA=/data/pgsql/17/data/|' \
  /usr/lib/systemd/system/postgresql-17.service

# Verify
grep PGDATA /usr/lib/systemd/system/postgresql-17.service
# Expected: Environment=PGDATA=/data/pgsql/17/data/

sudo systemctl daemon-reload
```

---

## Phase 7 — Configure postgresql.conf

Replace the default postgresql.conf with the tuned configuration. Parameters are adjusted for **8GB RAM / 4 CPU** server with NVMe disks.

```bash
sudo -u postgres tee /data/pgsql/17/data/postgresql.conf << 'EOF'
#------------------------------------------------------------------------------
# FILE LOCATIONS
#------------------------------------------------------------------------------
data_directory = '/data/pgsql/17/data'

#------------------------------------------------------------------------------
# CONNECTIONS
#------------------------------------------------------------------------------
listen_addresses = '*'
port = 5432
max_connections = 1000
reserved_connections = 10
superuser_reserved_connections = 3

#------------------------------------------------------------------------------
# MEMORY (8GB RAM, 4 CPU)
#------------------------------------------------------------------------------
shared_buffers = 2GB
huge_pages = try
temp_buffers = 64MB
max_prepared_transactions = 0
work_mem = 8MB
hash_mem_multiplier = 2.0
maintenance_work_mem = 256MB
logical_decoding_work_mem = 64MB
max_stack_depth = 2MB
shared_memory_type = mmap
dynamic_shared_memory_type = posix
vacuum_buffer_usage_limit = 4MB

#------------------------------------------------------------------------------
# I/O (NVMe disks)
#------------------------------------------------------------------------------
effective_io_concurrency = 200
maintenance_io_concurrency = 100
max_worker_processes = 3
max_parallel_workers_per_gather = 2
max_parallel_maintenance_workers = 2
max_parallel_workers = 3

#------------------------------------------------------------------------------
# WAL
#------------------------------------------------------------------------------
wal_level = replica
fsync = on
synchronous_commit = local
wal_sync_method = fsync
full_page_writes = on
wal_compression = off
checkpoint_timeout = 10min
checkpoint_completion_target = 0.9
checkpoint_flush_after = 256kB
checkpoint_warning = 30s
max_wal_size = 8GB
min_wal_size = 512MB

#------------------------------------------------------------------------------
# ARCHIVING
#------------------------------------------------------------------------------
archive_mode = on
archive_command = 'test ! -f /logs/archive/%f && cp %p /logs/archive/%f'
archive_timeout = 0

#------------------------------------------------------------------------------
# REPLICATION
#------------------------------------------------------------------------------
max_wal_senders = 10
max_replication_slots = 10
wal_keep_size = 512
max_slot_wal_keep_size = 500
wal_sender_timeout = 180s
hot_standby = on
max_standby_archive_delay = 600s
max_standby_streaming_delay = 600s
hot_standby_feedback = on
wal_receiver_timeout = 180s
wal_retrieve_retry_interval = 5s

#------------------------------------------------------------------------------
# QUERY TUNING
#------------------------------------------------------------------------------
seq_page_cost = 1.0
random_page_cost = 1.1
effective_cache_size = 3GB
enable_parallel_hash = on
enable_partition_pruning = on

#------------------------------------------------------------------------------
# LOGGING
#------------------------------------------------------------------------------
log_destination = 'stderr'
logging_collector = on
log_directory = '/logs/postgresql'
log_filename = 'pg-db1-%Y-%m-%d_%H%M%S.log'
log_file_mode = 0600
log_rotation_age = 1d
log_rotation_size = 50MB
log_truncate_on_rotation = off
log_min_messages = warning
log_min_error_statement = error
log_min_duration_statement = 3000
log_checkpoints = on
log_connections = off
log_disconnections = off
log_duration = on
log_error_verbosity = default
log_hostname = on
log_line_prefix = '%m [%p]: [%l-1] db=%d,user=%u,app=%a,client=%h '
log_lock_waits = on
log_statement = 'ddl'
log_temp_files = 1024
log_timezone = 'UTC'
syslog_facility = 'LOCAL0'
syslog_ident = 'postgres'

#------------------------------------------------------------------------------
# AUTOVACUUM
#------------------------------------------------------------------------------
autovacuum = on
autovacuum_max_workers = 3
autovacuum_naptime = 5min
autovacuum_vacuum_threshold = 500
autovacuum_vacuum_insert_threshold = 1000
autovacuum_analyze_threshold = 500
autovacuum_vacuum_scale_factor = 0.2
autovacuum_vacuum_insert_scale_factor = 0.2
autovacuum_analyze_scale_factor = 0.1
autovacuum_freeze_max_age = 200000000
autovacuum_multixact_freeze_max_age = 400000000
autovacuum_vacuum_cost_delay = 2ms
vacuum_cost_delay = 0
vacuum_cost_limit = 600
bgwriter_delay = 200ms
bgwriter_lru_maxpages = 1000
bgwriter_lru_multiplier = 7.0

#------------------------------------------------------------------------------
# CLIENT DEFAULTS
#------------------------------------------------------------------------------
statement_timeout = 0
lock_timeout = 5min
idle_in_transaction_session_timeout = 5min
idle_session_timeout = 10min
vacuum_freeze_table_age = 150000000
vacuum_freeze_min_age = 50000000
vacuum_failsafe_age = 1600000000
datestyle = 'iso, mdy'
timezone = 'UTC'
lc_messages = 'en_US.UTF-8'
lc_monetary = 'en_US.UTF-8'
lc_numeric = 'en_US.UTF-8'
lc_time = 'en_US.UTF-8'
default_text_search_config = 'pg_catalog.english'

#------------------------------------------------------------------------------
# LOCK MANAGEMENT
#------------------------------------------------------------------------------
deadlock_timeout = 1s
max_locks_per_transaction = 256
max_pred_locks_per_transaction = 256

#------------------------------------------------------------------------------
# PRELOAD LIBRARIES
#------------------------------------------------------------------------------
shared_preload_libraries = 'pg_stat_statements'
max_files_per_process = 5000
EOF
```

### Key Parameter Decisions

| Parameter | Value | Reason |
|-----------|-------|--------|
| `shared_buffers` | 2GB | 25% of 8GB RAM. Senior DBA config had 14GB (for 32GB server) — that would crash this server. |
| `effective_cache_size` | 3GB | Tells planner how much OS cache is available. Set conservatively given MongoDB also runs here. |
| `random_page_cost` | 1.1 | NVMe drives have nearly equal random/sequential speed. Default 4.0 is for HDD. Lower value tells planner to prefer index scans. |
| `max_connections` | 1000 | Confirmed by senior DBA. |
| `max_prepared_transactions` | 0 | Disabled — not used in this application. |
| `max_worker_processes` | 3 | Confirmed by senior DBA. Matches CPU count. |
| `log_statement` | ddl | Logs only DDL (CREATE/ALTER/DROP). Senior DBA config had `all` which logs every SELECT — would fill disk quickly. |
| `archive_mode` | on | Required for WAL archiving and backup tools. |
| `archive_command` | cp to /logs/archive | Local archive. Update this when backup server is configured with pgBackRest. |
| `wal_level` | replica | Minimum level required for streaming replication and backup tools. |
| `max_wal_size` | 8GB | Confirmed by senior DBA. Enough space on /data disk (47GB free). |

---

## Phase 8 — Configure pg_hba.conf

This file controls who can connect to PostgreSQL, from where, and how they authenticate.

> **Important:** Do NOT keep the default `trust` authentication entries. `trust` allows passwordless access to any local user including superuser — dangerous in production.

```bash
sudo tee /data/pgsql/17/data/pg_hba.conf << 'EOF'
# PostgreSQL Client Authentication Configuration File
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# Local socket — postgres OS user (OS-level peer authentication, no password needed)
local   all             postgres                                peer

# Local socket — all other users require password
local   all             all                                     scram-sha-256

# Localhost IPv4
host    all             all             127.0.0.1/32            scram-sha-256

# Localhost IPv6
host    all             all             ::1/128                 scram-sha-256

# Replication connections — localhost only
local   replication     all                                     peer
host    replication     all             127.0.0.1/32            scram-sha-256
host    replication     all             ::1/128                 scram-sha-256

# DBA access via VPN (pgAdmin from admin laptop)
host    all             all             10.0.17.0/24            scram-sha-256

# Application/Backend server
host    all             all             10.0.5.221/32           scram-sha-256

# Backup server — add IP when available
# host    all             all             BACKUP_SERVER_IP/32     scram-sha-256
EOF
```

### pg_hba.conf Explanation

| Entry | Purpose |
|-------|---------|
| `local postgres peer` | DBA connecting via Unix socket as postgres OS user — no password needed because OS already verified identity |
| `local all scram-sha-256` | Any other local connection requires password |
| `127.0.0.1/32 scram-sha-256` | Localhost TCP connections require password |
| `10.0.17.0/24 scram-sha-256` | DBA VPN subnet — pgAdmin access. VPN assigns IPs in this range. |
| `10.0.5.221/32 scram-sha-256` | Application/backend server IP — allows backend to connect |

> **Note on authentication method:** `scram-sha-256` is the most secure password authentication method available in PostgreSQL. Never use `md5` or `trust` in production.

> **Finding your VPN IP:** When you get a "no pg_hba.conf entry for host X.X.X.X" error in pgAdmin, X.X.X.X is your VPN IP. Add the corresponding subnet (e.g., `10.0.17.0/24`) to pg_hba.conf.

> **Common mistake:** Do not put a subnet (e.g., `10.0.17.0/24`) in pgAdmin's Host field. The Host field requires a specific IP or hostname of the DB server (e.g., `10.0.130.206`).

---

## Phase 9 — Set postgres User Password

```bash
sudo -u postgres psql
```

```sql
ALTER USER postgres WITH PASSWORD 'your_strong_password_here';
\q
```

---

## Phase 10 — Enable and Start PostgreSQL 17

```bash
sudo systemctl enable postgresql-17
sudo systemctl start postgresql-17
sudo systemctl status postgresql-17
```

Expected: `Active: active (running)`

---

## Phase 11 — Verification

```bash
sudo -u postgres psql -c "SHOW data_directory; SHOW log_directory; SHOW archive_mode;"
```

Expected output:
```
   data_directory
---------------------
 /data/pgsql/17/data

  log_directory
------------------
 /logs/postgresql

 archive_mode
--------------
 on
```

Verify log file is being written:
```bash
ls -la /logs/postgresql/
# Should see a pg-db1-YYYY-MM-DD_HHMMSS.log file
```

Verify WAL archive is working:
```bash
sudo -u postgres psql -c "SELECT pg_switch_wal();"
ls /logs/archive/
# Should see a WAL segment file appear
```

Verify pg_stat_statements extension:
```bash
sudo -u postgres psql -c "CREATE EXTENSION IF NOT EXISTS pg_stat_statements;"
sudo -u postgres psql -c "SELECT count(*) FROM pg_stat_statements;"
```

---

## Optional — Add Swap

This server has 0 swap. Production servers should have swap to prevent OOM kills.

```bash
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

---

## Troubleshooting Reference

### PostgreSQL fails to start — Permission denied on log or data directory

Cause: SELinux blocking access to custom directories.

Fix:
```bash
# Check SELinux context
ls -laZ /data/pgsql/17/data/
ls -laZ /logs/postgresql/

# If context is unlabeled_t or wrong type, fix parent first
sudo semanage fcontext -a -t var_t "/data(/.*)?"
sudo restorecon -Rv /data
sudo semanage fcontext -a -t var_log_t "/logs(/.*)?"
sudo restorecon -Rv /logs

# Then apply specific PostgreSQL contexts
sudo semanage fcontext -a -t postgresql_db_t "/data/pgsql(/.*)?"
sudo semanage fcontext -a -t postgresql_log_t "/logs/postgresql(/.*)?"
sudo semanage fcontext -a -t postgresql_log_t "/logs/archive(/.*)?"
sudo chcon -Rv -t postgresql_db_t /data/pgsql
sudo chcon -Rv -t postgresql_log_t /logs/postgresql
sudo chcon -Rv -t postgresql_log_t /logs/archive
```

### pgAdmin — "no pg_hba.conf entry for host X.X.X.X"

Cause: The connecting IP is not listed in pg_hba.conf.

Fix: Add the IP or subnet to pg_hba.conf, then reload:
```bash
sudo -u postgres psql -c "SELECT pg_reload_conf();"
```

### pgAdmin — "failed to resolve host 'X.X.X.X/24'"

Cause: A subnet was entered in pgAdmin's Host field instead of a specific IP.

Fix: The Host field must be the DB server's IP (e.g., `10.0.130.206`), not a subnet.

### dnf module disable fails

If `dnf module disable postgresql -y` does not work cleanly:
```bash
dnf module list | grep postgres
dnf module disable postgresql:16  # or whichever version is listed
dnf list installed | grep postgresql
dnf remove postgresql*
dnf clean all
dnf makecache
```

---

## Post-Setup Checklist

- [ ] PG 16 completely removed (`rpm -qa | grep postgresql` returns empty)
- [ ] PG 17 installed and version verified
- [ ] Custom directories created with correct ownership and SELinux context
- [ ] initdb completed successfully with `--data-checksums`
- [ ] systemd service file updated with correct PGDATA
- [ ] postgresql.conf applied with tuned parameters
- [ ] pg_hba.conf configured — no `trust` entries
- [ ] postgres user password set
- [ ] Service enabled and running
- [ ] data_directory, log_directory, archive_mode verified
- [ ] Log file appears in /logs/postgresql/
- [ ] WAL archive working — files appear in /logs/archive/
- [ ] pgAdmin connects successfully from DBA machine
