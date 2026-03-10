# PostgreSQL HA Cluster — Production Best Practices
## Three-Way Comparison: Our Lab Setup vs Stage Server vs Real Production Standard

---

## Table of Contents

1. [Component Versions](#1-component-versions)
2. [Directory and File Structure](#2-directory-and-file-structure)
3. [etcd Configuration](#3-etcd-configuration)
4. [PostgreSQL Configuration](#4-postgresql-configuration)
5. [Patroni Configuration](#5-patroni-configuration)
6. [HAProxy Configuration](#6-haproxy-configuration)
7. [Replication Strategy](#7-replication-strategy)
8. [WAL Archiving and Backup](#8-wal-archiving-and-backup)
9. [Security](#9-security)
10. [Monitoring and Alerting](#10-monitoring-and-alerting)
11. [Summary Scorecard](#11-summary-scorecard)

---

## 1. Component Versions

### Comparison Table

| Component | Our Lab Setup | Stage Server | Real Production Standard | Why |
|-----------|--------------|--------------|--------------------------|-----|
| PostgreSQL | 16 | 17.7 | Latest stable minor of current major. As of 2026: **17.x** | Minor versions contain only bug fixes and security patches — safe to upgrade with just a restart. Running outdated minor = known vulnerabilities unpatched. |
| Patroni | 4.1.0 | 4.1.0 | Latest stable. As of 2026: **4.1.x** | Bug fixes and new Patroni features directly affect failover reliability. |
| etcd | 3.5.17 | 3.5.0 | Latest stable patch of 3.5.x. As of 2026: **3.5.17+** | etcd 3.5.0 had a confirmed data inconsistency bug under crash scenarios. Fixed in 3.5.3. Running 3.5.0 in production is a known risk. |
| HAProxy | 2.8.x | System package | **2.8.x LTS** or **2.9.x** | LTS versions receive security patches longer. System package may lag behind. |
| OS | Rocky Linux 9 | Rocky Linux 9 | Rocky Linux 9 / RHEL 9 | RHEL-based OS has long-term support, stable kernel, and enterprise PostgreSQL repo compatibility. |

### Explanation

**PostgreSQL versioning:**
PostgreSQL releases one major version per year (e.g., 15, 16, 17). Within a major version, minor releases (e.g., 17.1, 17.2, 17.7) contain only bug fixes and security patches — never breaking changes. In production, always run the latest minor version of your chosen major version. Major version upgrades require `pg_upgrade` and planned downtime.

```
PostgreSQL 17.7 → "17" is major, "7" is minor
Always upgrade minor versions (17.1 → 17.7) — safe, just restart
Major upgrades (16 → 17) require planning and pg_upgrade
```

**etcd versioning:**
etcd 3.5.0 had a known data inconsistency bug under specific crash scenarios, fixed in 3.5.3. Production should always run 3.5.3 or later. Our lab setup uses 3.5.17 which is correct.

**Pros/Cons:**

| | Our Lab | Stage | Production Standard |
|--|---------|-------|---------------------|
| Security patches | ✅ Up to date (PG16) | ⚠️ etcd outdated | ✅ All latest |
| Bug fixes | ✅ | ⚠️ etcd 3.5.0 bug risk | ✅ |
| Stability | ✅ | ✅ Running 1+ months | ✅ |

---

## 2. Directory and File Structure

### Comparison Table

| Path | Purpose | Our Lab | Stage Server | Production Standard | Why |
|------|---------|---------|--------------|---------------------|-----|
| PostgreSQL data | Database files | `/data/patroni` | `/data/pgsql` | **Dedicated disk**, e.g. `/data/pgsql` or `/pgdata` | OS disk filling up (logs, tmp) must not crash PostgreSQL. Separate disk isolates failures. |
| etcd data | Cluster state | `/etcd/data` | `/data/etcd` | **Dedicated disk**, e.g. `/data/etcd` | etcd is latency-sensitive. Sharing disk with PostgreSQL causes I/O contention and can trigger false leader elections. |
| WAL archive | PITR backups | Not configured | `/log/archive/` (local) | **Remote storage**: NFS, S3, or dedicated backup server | Local archive is lost if the server dies — defeating the purpose of having an archive. |
| Patroni config | patroni.yml | `/etc/patroni/` | `/etc/patroni/` | `/etc/patroni/` ✅ standard | Standard path, consistent across nodes. |
| Patroni logs | Service logs | journald only | `/etc/patroni/logs/` | `/var/log/patroni/` or dedicated log volume | File logs persist across service restarts and are easier to grep. journald can lose old entries under rotation. |
| etcd config | etcd settings | `/etc/etcd/etcd.conf` | Command line args in systemd | Either is acceptable; **config file is more maintainable** | Config file is easier to read, diff, and manage with Ansible/Chef. Command line args in systemd get long and hard to maintain. |
| PostgreSQL bin | PG binaries | `/usr/pgsql-16/bin` | `/usr/pgsql-17/bin` | `/usr/pgsql-{version}/bin` | PGDG installs each major version in its own path — allows multiple versions to coexist. |
| Backup base | Physical backups | Not configured | Not observed | `/backup/base/` on **separate server or storage** | Backup on same server as data offers no protection against server failure or disk loss. |

### Production Standard — Disk Layout

In real production, the most critical rule is **separation of data across disks**:

```
/           → OS disk (50-100GB SSD)
/data/pgsql → PostgreSQL data (separate SSD/NVMe, sized for DB + 30% growth)
/data/etcd  → etcd data (separate disk or at minimum separate partition)
/data/wal   → Optional: pg_wal on separate disk for I/O isolation
/backup     → Backup target (NFS mount or object storage)
```

**Why separate disks?**

If PostgreSQL data and OS share one disk and the OS fills up (logs, temp files, etc.), PostgreSQL cannot write WAL and crashes. With separate disks, each failure is isolated.

**Our lab vs Stage vs Production:**

| | Our Lab | Stage | Production Standard |
|--|---------|-------|---------------------|
| PG data on dedicated disk | ❌ Not verified | ❌ Not verified | ✅ Required |
| etcd on dedicated disk | ❌ Not verified | ✅ `/data/etcd` separate | ✅ Required |
| Archive on remote storage | ❌ Not configured | ❌ Local disk only | ✅ Required |
| Patroni logs to file | ❌ journald only | ✅ `/etc/patroni/logs/` | ✅ Required |

---

## 3. etcd Configuration

### Comparison Table

| Parameter | Our Lab | Stage Server | Production Standard | Why |
|-----------|---------|--------------|---------------------|-----|
| Config style | File (`/etc/etcd/etcd.conf`) | Command line args | **File preferred** — more maintainable | File can be version-controlled, diffed, and managed by config management tools. Command line args in systemd become unreadable at scale. |
| Data directory | `/etcd/data` | `/data/etcd` | Dedicated disk path | Dedicated disk prevents I/O contention with PostgreSQL and OS. etcd is fsync-heavy — shared disk degrades its latency. |
| `--enable-v2` | Not set | `--enable-v2=true` | **Not set** unless another service requires it | etcd v2 API is deprecated and will be removed in etcd 3.6. Enabling it unnecessarily increases attack surface and future upgrade complexity. |
| `ETCD_QUOTA_BACKEND_BYTES` | 8GB | Not observed (default 2GB) | **8GB minimum** | Default 2GB fills up within months in a busy Patroni cluster. When quota is exceeded, etcd stops all writes — Patroni loses coordination and the cluster freezes. |
| `ETCD_AUTO_COMPACTION_MODE` | `revision` | Not observed | `revision` | Without compaction, etcd retains full history of every write forever. DB grows indefinitely until quota is hit. |
| `ETCD_AUTO_COMPACTION_RETENTION` | `1000` | Not observed | `1000` | Keeps last 1000 revisions. Old history beyond this is deleted automatically, keeping DB size stable. |
| Number of nodes | 3 | 3 | **3 or 5** (always odd) | Raft requires majority quorum. 3 nodes = tolerate 1 failure. 5 nodes = tolerate 2 failures. Even numbers risk split votes. |
| TLS encryption | No | No | **Yes** in production | Without TLS, anyone with network access can read or write etcd keys including the leader key — complete cluster compromise possible. |

### Production etcd Config Explained

**Quota (8GB):**
etcd stores every write as a new revision, building up history. Default quota is 2GB — in a busy Patroni cluster writing every 10 seconds, 2GB can fill up within months. When quota is exceeded, etcd stops accepting writes, Patroni loses coordination, and the cluster freezes. 8GB gives years of headroom.

```
Default 2GB quota:
- Patroni writes every 10s = ~6 writes/min = ~8640 writes/day
- Each write ~1KB = ~8.4MB/day
- 2GB fills in ~238 days without compaction
- With compaction: manageable, but 8GB is safer
```

**Auto Compaction:**
etcd keeps full revision history. Compaction deletes old revisions. `revision` mode with `1000` means keep last 1000 revisions, delete the rest. Without this, the database grows indefinitely.

**`--enable-v2=true`:**
etcd v2 API is deprecated and will be removed in etcd 3.6. Stage server has it enabled — likely because Kubernetes components on the same server need it. In a pure Patroni setup, this should not be set.

**TLS:**
In real production, etcd peer and client communication should use TLS certificates. Without TLS, anyone with network access can read/write the etcd keys, including the leader key. Our lab and stage both use plain HTTP — acceptable for internal isolated networks, not acceptable if the network is shared.

**Production etcd config file example:**
```ini
ETCD_NAME="node1"
ETCD_DATA_DIR="/data/etcd"
ETCD_LISTEN_PEER_URLS="https://192.168.1.1:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.1.1:2379,https://127.0.0.1:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.1.1:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.1.1:2379"
ETCD_INITIAL_CLUSTER="node1=https://192.168.1.1:2380,node2=https://192.168.1.2:2380,node3=https://192.168.1.3:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="prod-etcd-cluster"
ETCD_QUOTA_BACKEND_BYTES=8589934592
ETCD_AUTO_COMPACTION_MODE=revision
ETCD_AUTO_COMPACTION_RETENTION=1000
ETCD_PEER_CERT_FILE="/etc/etcd/certs/peer.crt"
ETCD_PEER_KEY_FILE="/etc/etcd/certs/peer.key"
ETCD_PEER_TRUSTED_CA_FILE="/etc/etcd/certs/ca.crt"
ETCD_CERT_FILE="/etc/etcd/certs/server.crt"
ETCD_KEY_FILE="/etc/etcd/certs/server.key"
ETCD_CLIENT_CERT_AUTH="true"
```

**Pros/Cons:**

| | Our Lab | Stage | Production Standard |
|--|---------|-------|---------------------|
| Quota set correctly | ✅ 8GB | ⚠️ Likely default 2GB | ✅ 8GB+ |
| Auto compaction | ✅ | ⚠️ Not observed | ✅ |
| TLS | ❌ | ❌ | ✅ |
| Config maintainability | ✅ File based | ⚠️ Command line | ✅ File based |

---

## 4. PostgreSQL Configuration

### Memory Parameters

| Parameter | Our Lab | Stage Server | Production Standard | Explanation |
|-----------|---------|--------------|---------------------|-------------|
| `shared_buffers` | Default (128MB) | 4GB | **25% of total RAM** | PostgreSQL's main cache. Most important parameter. |
| `effective_cache_size` | Default (4GB) | 10GB | **75% of total RAM** | Tells query planner how much OS cache is available. Does not allocate memory — only a hint. |
| `work_mem` | Default (4MB) | 8MB | **RAM / (max_connections × 2)** | Memory per sort/hash operation. Too high risks OOM with many connections. |
| `maintenance_work_mem` | Default (64MB) | Not set | **1-2GB** | Memory for VACUUM, CREATE INDEX, pg_restore. Higher = faster maintenance. |
| `max_connections` | Default (100) | 1000 | **Based on workload**; use PgBouncer for high connection counts | Each idle connection uses ~5-10MB. 1000 connections = up to 10GB just for connections. |
| `max_wal_size` | Default (1GB) | 1GB | **2-4GB** for busy systems | Maximum WAL size between checkpoints. Too small = frequent checkpoints = I/O spikes. |
| `checkpoint_completion_target` | Default (0.9) | Not set | **0.9** | Spreads checkpoint I/O over 90% of checkpoint interval. Reduces I/O spikes. |
| `wal_buffers` | Default (-1 = auto) | Not set | **64MB** | WAL write buffer. -1 = 1/32 of shared_buffers. Explicit 64MB recommended. |
| `random_page_cost` | Default (4.0) | Not set | **1.1** for SSD | Tells planner cost of random I/O. 4.0 assumes spinning disk. SSD is ~1.1. Wrong value leads to bad query plans. |
| `effective_io_concurrency` | Default (1) | Not set | **200** for SSD | How many concurrent I/O operations the planner assumes. 200 for SSD, 2-4 for HDD. |

**Sizing example for a server with 32GB RAM:**
```yaml
shared_buffers: '8GB'               # 25% of 32GB
effective_cache_size: '24GB'        # 75% of 32GB
work_mem: '16MB'                    # 32GB / (1000 connections × 2)
maintenance_work_mem: '2GB'
wal_buffers: '64MB'
max_wal_size: '4GB'
checkpoint_completion_target: 0.9
random_page_cost: 1.1               # assuming SSD
effective_io_concurrency: 200       # assuming SSD
```

### WAL and Replication Parameters

| Parameter | Our Lab | Stage Server | Production Standard | Explanation |
|-----------|---------|--------------|---------------------|-------------|
| `wal_level` | `replica` | `hot_standby` (→ `replica`) | **`replica`** | Minimum for streaming replication. `hot_standby` is an old alias, still works. |
| `max_wal_senders` | 10 | 10 | **Number of replicas + 3** for headroom | One sender process per replica connection. Extra for pg_basebackup and monitoring. |
| `max_replication_slots` | 10 | 10 | **Number of replicas + 3** | One slot per replica. Extra for backup tools. |
| `wal_keep_size` | Not set | 1600MB (auto-converted) | **1GB minimum** | WAL retained even without slots. Safety net if slots are dropped. |
| `hot_standby` | `on` | `on` | **`on`** | Allows read queries on replicas. |
| `hot_standby_feedback` | Not set | Not set | **`on`** | Replica tells primary about long-running queries to prevent vacuum conflicts. |
| `wal_log_hints` | Not set | Not set | **`on`** | Required for `pg_rewind`. Without this, pg_rewind cannot resync a diverged node. |

**Note on `wal_log_hints`:**
Stage server has `use_pg_rewind: true` in Patroni config but `wal_log_hints` is not explicitly set. This works only because `data-checksums` was enabled at initdb time (checksums imply hint bits logging). In production, explicitly set `wal_log_hints: on` to avoid dependency on checksums being present.

### Autovacuum Parameters

These are not set in our lab or stage but are critical in production:

| Parameter | Default | Production Recommendation | Explanation |
|-----------|---------|---------------------------|-------------|
| `autovacuum_max_workers` | 3 | **5-6** | More workers for larger databases |
| `autovacuum_vacuum_scale_factor` | 0.2 (20%) | **0.01-0.05** for large tables | Default triggers vacuum after 20% of table is dead tuples. A 100M row table would need 20M dead rows before vacuuming. Set lower for large tables. |
| `autovacuum_analyze_scale_factor` | 0.1 (10%) | **0.01** for large tables | Same issue for ANALYZE |
| `autovacuum_vacuum_cost_delay` | 2ms | **0** for SSDs | Throttles vacuum I/O. On SSD, vacuum can run full speed. |

**Why autovacuum matters:**
PostgreSQL uses MVCC — old row versions (dead tuples) are not deleted immediately. Autovacuum cleans them up. If autovacuum falls behind, tables bloat (wasted disk space), query performance degrades, and eventually transaction ID wraparound can cause database shutdown.

**Pros/Cons:**

| | Our Lab | Stage | Production Standard |
|--|---------|-------|---------------------|
| Memory tuned | ❌ Default | ✅ Tuned for server | ✅ Tuned per workload |
| SSD-optimized | ❌ | ❌ Not observed | ✅ |
| Autovacuum tuned | ❌ | ❌ Not observed | ✅ |
| `wal_log_hints` explicit | ❌ | ❌ | ✅ |

---

## 5. Patroni Configuration

### Core DCS Settings

| Parameter | Our Lab | Stage Server | Production Standard | Why |
|-----------|---------|--------------|---------------------|-----|
| `ttl` | 30 | 30 | **30** | TTL too low (e.g. 10s) causes false failovers on transient network blips. Too high (e.g. 120s) means 2 minutes of downtime after a real crash. 30s is the proven balance. |
| `loop_wait` | 10 | 10 | **10** | Should be ttl/3 so the primary has 3 chances to renew before TTL expires. Too low wastes CPU; too high means slow health detection. |
| `retry_timeout` | 10 | 10 | **10** | Covers transient etcd or PostgreSQL unavailability without immediately triggering a failover. Should match loop_wait. |
| `maximum_lag_on_failover` | 1048576 (1MB) | 1048576 (1MB) | **1048576 (1MB)** | Prevents promoting a replica that is significantly behind. A heavily lagged replica becoming primary means data loss proportional to the lag. |
| `master_start_timeout` | Not set | Not set | **300** | Without this, if a primary takes more than `loop_wait` to start (e.g. crash recovery replaying WAL), Patroni may declare it failed and trigger unnecessary failover. |
| `synchronous_mode` | Not set | Not set | **`quorum`** if zero data loss required | Patroni-level enforcement of sync replication. Coordinates with PostgreSQL's `synchronous_standby_names` to ensure failover only promotes a confirmed sync replica. |

### PostgreSQL Authentication in Patroni

| Item | Our Lab | Stage Server | Production Standard | Why |
|------|---------|--------------|---------------------|-----|
| Superuser password | `postgres123` (weak) | `#PgThey63084nM#` (strong) | **Strong random password, stored in vault** | Superuser has unrestricted access to all databases. A weak or leaked password means full data compromise. |
| Replication password | `replicator123` | `replicator` (same as username — very weak) | **Strong random password** | Replication user can stream the entire database contents off the primary. This credential must be treated as high-value. |
| Password storage | Plaintext in yml | Plaintext in yml | **Encrypted using Ansible Vault, HashiCorp Vault, or environment variables** | Config files are often shared, version-controlled, or readable by ops teams. Plaintext credentials in these files are routinely leaked. |

**Production password approach:**
```yaml
# Instead of plaintext:
postgresql:
  authentication:
    superuser:
      username: postgres
      password: "{{ lookup('env', 'PG_SUPERUSER_PASSWORD') }}"
    replication:
      username: replicator
      password: "{{ lookup('env', 'PG_REPLICATION_PASSWORD') }}"
```

### Patroni Log Configuration

| Item | Our Lab | Stage Server | Production Standard | Why |
|------|---------|--------------|---------------------|-----|
| Log destination | journald only | `/etc/patroni/logs/` | `/var/log/patroni/` on dedicated log volume | journald entries can be lost after rotation or system restart. File logs persist and can be shipped to a central log system (ELK, Loki). |
| Log rotation | journald handles | `file_num: 5` | `file_num: 10`, external logrotate | 5 files may not cover enough history to diagnose an incident that happened days ago. 10 files with size limits gives better coverage. |
| Log level | Default | INFO | **INFO** (DEBUG only when troubleshooting) | DEBUG logs are extremely verbose — they can fill disk within hours on a busy cluster. INFO gives enough detail for incident diagnosis without the volume. |

**Production log config:**
```yaml
log:
  level: INFO
  traceback_level: INFO
  dir: /var/log/patroni
  file_num: 10
  file_size: 52428800  # 50MB per file
```

### tags Configuration

| Tag | Our Lab | Stage Server | Explanation |
|-----|---------|--------------|-------------|
| `nofailover` | false | false | `true` = this node will never be promoted. Use for a node on weaker hardware or in a remote DC. |
| `noloadbalance` | false | false | `true` = HAProxy excludes this node from read traffic. Use during maintenance. |
| `clonefrom` | false | false | `true` = new nodes prefer to clone from this node. Use on the node with fastest disk. |
| `nosync` | false | false | `true` = this node is never chosen as synchronous standby. Use for a distant replica where sync would add too much latency. |

**Pros/Cons:**

| | Our Lab | Stage | Production Standard |
|--|---------|-------|---------------------|
| Password strength | ❌ Weak | ✅ Superuser strong, ❌ replication weak | ✅ All strong |
| Password encryption | ❌ Plaintext | ❌ Plaintext | ✅ Vault/env vars |
| Log to file | ❌ | ✅ | ✅ |
| `master_start_timeout` set | ❌ | ❌ | ✅ |

---

## 6. HAProxy Configuration

### Comparison Table

| Parameter | Our Lab | Stage Server | Production Standard | Why |
|-----------|---------|--------------|---------------------|-----|
| Primary health endpoint | `/primary` | `/master` (deprecated) | **`/primary`** | `/master` will be removed in a future Patroni version. When that happens, HAProxy marks all nodes DOWN and the cluster becomes unreachable — silent outage. |
| Stats authentication | None | `admin:admin123` (weak) | **Strong password or IP-restricted** | Stats dashboard exposes server states, connection counts, and traffic — useful intel for an attacker. |
| `maxconn` global | 100 | 500 | **Match PostgreSQL `max_connections`** | If HAProxy's maxconn is lower than PostgreSQL's, it becomes the bottleneck before PostgreSQL is even stressed. |
| `maxconn` per server | 100 | 1000 | **Match PostgreSQL `max_connections`** | Per-server limit must match the global limit. Mismatched values cause unexpected connection rejections. |
| `timeout client` | 30m | 60m | **Based on longest expected query** | Too short = long-running reports get disconnected mid-query. Too long = dead connections accumulate and hold server resources. |
| `balance` algorithm | Default (roundrobin) | `roundrobin` explicit | **`roundrobin`** for replicas; N/A for primary (only 1 UP) | roundrobin distributes read load evenly. Only one server is ever UP in the primary backend so balance has no effect there. |
| HAProxy redundancy | Single instance | Single instance | **2× HAProxy + Keepalived + Virtual IP** | Single HAProxy is a SPOF — if it crashes, the application loses database connectivity even though the DB cluster is healthy. |
| SSL termination | None | None | **SSL at HAProxy** if clients are outside internal network | Without SSL, passwords and query data travel in plaintext over the network — interception risk. |
| `on-marked-down` | `shutdown-sessions` | `shutdown-sessions` | **`shutdown-sessions`** ✅ | Without this, existing connections to a demoted primary stay open and receive errors. Closing them immediately forces the application to reconnect to the new primary. |

### HAProxy Single Point of Failure

Both our lab and stage have a single HAProxy server. If HAProxy goes down, the application loses its database endpoint even though the database cluster is healthy.

**Production solution — Keepalived + Virtual IP:**

```
                    Virtual IP: 192.168.1.100
                         ↓
              ┌──────────┴──────────┐
              │                     │
         HAProxy1               HAProxy2
         (MASTER)               (BACKUP)
         192.168.1.10           192.168.1.11
```

Keepalived monitors both HAProxy instances. If HAProxy1 fails, the Virtual IP moves to HAProxy2 within 2-3 seconds. The application always connects to the Virtual IP and never notices the HAProxy switch.

**Production HAProxy config additions:**
```
frontend pg_write
    bind *:5000
    # Add client certificate verification if needed
    default_backend pg_primary

frontend pg_read
    bind *:5001
    default_backend pg_replicas

backend pg_primary
    option httpchk GET /primary
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server node1 192.168.1.1:5432 maxconn 500 check port 8008
    server node2 192.168.1.2:5432 maxconn 500 check port 8008
    server node3 192.168.1.3:5432 maxconn 500 check port 8008
```

**Note — `frontend`/`backend` vs `listen`:**
Stage uses `listen` blocks which combine frontend and backend. Production typically uses separate `frontend` and `backend` blocks for more flexibility — e.g., one frontend can route to different backends based on ACLs.

**Pros/Cons:**

| | Our Lab | Stage | Production Standard |
|--|---------|-------|---------------------|
| Correct health endpoint | ✅ `/primary` | ⚠️ `/master` deprecated | ✅ `/primary` |
| Stats auth | ❌ None | ⚠️ Weak password | ✅ Strong or IP-restricted |
| HAProxy redundancy | ❌ Single | ❌ Single | ✅ Dual + Keepalived |
| SSL | ❌ | ❌ | ✅ If clients outside internal network |

---

## 7. Replication Strategy

### Comparison Table

| Aspect | Our Lab | Stage Server | Production Standard | Why |
|--------|---------|--------------|---------------------|-----|
| Replication type | Async (default) | Quorum sync (`ANY 1`) | **Depends on RPO requirement** | Async = fast writes but data loss possible on crash. Sync = zero data loss but slightly slower writes. Choice depends on how much data loss is acceptable. |
| `synchronous_commit` | Default (`on` locally, no remote sync) | `on` | **`on` with remote sync** for zero data loss | `synchronous_commit=on` alone without `synchronous_standby_names` only ensures local WAL flush — not replica confirmation. Both must be set for true zero data loss. |
| `synchronous_standby_names` | Not set | `ANY 1 (node1, node2, node3)` | `ANY 1 (node2, node3)` or `FIRST 1 (node2)` | `ANY 1` means any one replica must confirm before commit. More resilient than `FIRST 1` — writes continue as long as at least one replica is reachable. |
| Replication slots | `use_slots: true` | `use_slots: true` | **`use_slots: true`** ✅ | Slots guarantee primary retains WAL that a replica still needs. Without slots, if a replica falls behind, the primary may delete WAL it needs and the replica must do a full resync. |
| `pg_rewind` | `use_pg_rewind: true` | `use_pg_rewind: true` | **`use_pg_rewind: true`** ✅ | After a failover, the old primary's WAL has diverged. Without pg_rewind, it must do a full resync (copying entire dataset). pg_rewind only copies the diverged portion — much faster. |
| `hot_standby_feedback` | Not set | Not set | **`on`** recommended | Without this, the primary's autovacuum can delete row versions that a long-running replica query still needs, causing `conflict with recovery` errors and query cancellations on the replica. |

### Replication Modes Explained

**Async replication:**
```
Primary → Commits → Confirms to client → Sends WAL to replica (background)
```
- Fastest write performance
- Risk: If primary crashes before WAL reaches replica, last transactions are lost
- RPO (Recovery Point Objective): seconds to minutes of data loss possible

**Synchronous replication (`FIRST 1`):**
```
Primary → Waits for node2 to confirm WAL received → Commits → Confirms to client
```
- node2 must always be up and fast
- If node2 is slow or unavailable, all writes on primary stall
- RPO: Zero data loss

**Quorum sync (`ANY 1`) — Stage server's approach:**
```
Primary → Waits for any ONE of (node2, node3) to confirm → Commits → Confirms to client
```
- More resilient than FIRST 1: if node2 is down, node3 can serve as the sync replica
- Writes only stall if ALL replicas are unreachable
- RPO: Zero data loss as long as at least one replica is up
- This is the best balance for a 3-node cluster

**When to use what:**

| Scenario | Recommendation |
|----------|---------------|
| OLTP, financial data, cannot lose transactions | `ANY 1 (replica1, replica2)` quorum |
| Analytics, can tolerate small data loss for speed | Async |
| Reporting replica in another datacenter | Async (latency too high for sync) |
| DR replica in remote site | Async with `nosync: true` tag |

**Replication Slots — Important Consideration:**

Slots prevent the primary from deleting WAL that a replica still needs. This is safe and necessary. However:

```
Replica goes offline → slot stays active → primary retains ALL WAL since replica disconnected
→ If replica is offline for days, primary's pg_wal directory can fill the disk
→ When disk is full, primary crashes
```

**Production mitigation:**
```yaml
# In patroni DCS config
postgresql:
  parameters:
    max_slot_wal_keep_size: '10GB'  # Limit WAL retained per slot
```
If a replica falls more than 10GB behind, the slot is dropped. The replica must do a full resync but the primary won't run out of disk.

**Pros/Cons:**

| | Our Lab | Stage | Production Standard |
|--|---------|-------|---------------------|
| Data loss risk | ⚠️ Async, small risk | ✅ Zero (quorum) | ✅ Zero (quorum) |
| Write performance | ✅ Fastest | ⚠️ Slightly slower | ⚠️ Slightly slower (acceptable) |
| Resilience to replica failure | ✅ Writes always succeed | ✅ Needs 1 of 2 replicas | ✅ Needs 1 of N replicas |
| `max_slot_wal_keep_size` | ❌ Not set | ❌ Not set | ✅ Required |

---

## 8. WAL Archiving and Backup

### Comparison Table

| Aspect | Our Lab | Stage Server | Production Standard | Why |
|--------|---------|--------------|---------------------|-----|
| `archive_mode` | off | on | **on** | Without archiving, PITR is impossible. If data is accidentally deleted, you can only restore to the last full backup — losing everything since then. |
| `archive_command` | Not set | `cp -f %p /log/archive/%f` (local) | **Remote**: `rsync`, `wal-g`, `pgbackrest` | Local archive is lost when the server dies. Remote archive survives server failure — the only kind of backup that actually protects against server loss. |
| `archive_timeout` | Not set | 600s (10 min) | **300s** (5 min) maximum data loss window | Without this, a WAL file is only archived when full. On a low-write system, that could take hours — meaning PITR cannot recover more recently than the last archived WAL. |
| `restore_command` | Not set | `cp /log/archive/%f %p` | Matching remote fetch command | Required for PITR. PostgreSQL uses this to fetch archived WAL during recovery. Without it, recovery stops at the base backup. |
| Physical backup | Not configured | Not observed | Daily `pg_basebackup` or `pgbackrest` | WAL archive alone is not enough — you need a base backup as the starting point for any restore. Without it, WAL is useless. |
| Logical backup | Not configured | Not observed | Weekly `pg_dump` | Logical backups allow restoring a single table or database without restoring the entire cluster. Useful for accidental DROP TABLE recovery. |
| Backup testing | N/A | N/A | **Monthly restore test** — critical | An untested backup is an assumption, not a recovery plan. Backup files can be corrupt, incomplete, or incompatible — you only find out during a real disaster if you never test. |
| Backup retention | N/A | N/A | 7 days physical + WAL, 30 days logical | Short retention risks not having a backup before a problem that was introduced days ago. Long retention wastes storage. 7 days physical covers most operational mistakes. |
| Backup monitoring | N/A | N/A | Alert if backup older than 25 hours | Backup jobs can silently fail. Without monitoring, you discover the backup was broken only when you need it most. |

### Archive Storage Options

**Local disk (Stage server approach) — Not recommended for production:**
- If the server dies, archive dies with it
- `pg_wal` and archive on same filesystem risks one filling the other

**NFS mount:**
```bash
archive_command: 'rsync -a %p backup-server:/archives/pg_wal/%f'
restore_command: 'rsync -a backup-server:/archives/pg_wal/%f %p'
```

**Object storage with WAL-G (Recommended for production):**
```bash
archive_command: 'wal-g wal-push %p'
restore_command: 'wal-g wal-fetch %f %p'
```
WAL-G compresses WAL files (typically 10:1 ratio), uploads to S3/GCS/Azure, and handles retention automatically.

**pgBackRest (Enterprise-grade):**
```bash
archive_command: 'pgbackrest --stanza=main archive-push %p'
restore_command: 'pgbackrest --stanza=main archive-get %f %p'
```
pgBackRest handles full backups, incremental backups, WAL archiving, PITR, and parallel restore — all in one tool.

### PITR Explained

Point-in-Time Recovery is the ability to restore a database to any past moment, not just the last backup.

```
Monday 00:00  → Full backup taken
Tuesday 14:30 → Developer runs: DELETE FROM orders WHERE 1=1;  ← Disaster
Tuesday 14:31 → You realize the mistake

Without PITR: Restore Monday 00:00 backup → lose 14.5 hours of data
With PITR:    Restore Monday 00:00 backup + apply WAL until 14:29:59 → lose 1 minute
```

**Requirements for PITR:**
1. `archive_mode = on`
2. `archive_command` that successfully copies WAL files
3. A base backup taken before the point you want to restore to
4. All WAL files between the base backup and target time

**Pros/Cons:**

| | Our Lab | Stage | Production Standard |
|--|---------|-------|---------------------|
| PITR possible | ❌ | ⚠️ Archive configured but local | ✅ Remote archive |
| Backup off-server | ❌ | ❌ | ✅ Required |
| Automated backup testing | ❌ | ❌ Not observed | ✅ Monthly |
| Backup tool | None | cp command | ✅ pgBackRest or WAL-G |

---

## 9. Security

### pg_hba Configuration

| Rule type | Our Lab | Stage Server | Production Standard | Why |
|-----------|---------|--------------|---------------------|-----|
| Replication access | Subnet `/24` | Per-node `/32` | **Per-node `/32`** | `/24` allows any machine on the subnet to impersonate a replica. `/32` restricts to exactly the three known node IPs. |
| Application access | `0.0.0.0/0 md5` | HAProxy IP + open `0.0.0.0` entries | **Only HAProxy IP `/32`** | Application should only reach PostgreSQL through HAProxy. Direct access bypasses load balancing and connection control. |
| Superuser remote access | Allowed from subnet | 🚨 Allowed from anywhere | **Blocked or only from bastion IP** | Superuser can drop databases, modify users, read all data. Remote superuser access is a critical attack surface. |
| Open `0.0.0.0/0` rules | ❌ Not present | 🚨 Present for postgres, all | **Never** | `0.0.0.0/0` means the entire internet if the port is reachable. Even with strong passwords, this enables brute force attacks from anywhere. |
| Authentication method | `md5` | `md5` | **`scram-sha-256`** — md5 is deprecated | MD5 is cryptographically weak — vulnerable to offline dictionary attacks if the hash is captured. SCRAM-SHA-256 is resistant to replay attacks and brute force. |

**`scram-sha-256` vs `md5`:**
`md5` hashes passwords with MD5 — weak by modern standards, vulnerable to offline brute force. `scram-sha-256` uses SCRAM protocol — resistant to replay attacks and much stronger. PostgreSQL 14+ defaults to scram-sha-256 for new clusters.

**Production pg_hba should look like:**
```
# Replication — only from known replica IPs
host  replication  replicator  192.168.1.2/32  scram-sha-256
host  replication  replicator  192.168.1.3/32  scram-sha-256

# Application — only from HAProxy
host  all  appuser        192.168.1.10/32  scram-sha-256
host  all  reporting_user 192.168.1.10/32  scram-sha-256

# DBA access — only from bastion/jump server
host  all  postgres  192.168.1.50/32  scram-sha-256

# Patroni internal — only from cluster nodes
host  all  postgres  192.168.1.1/32  scram-sha-256
host  all  postgres  192.168.1.2/32  scram-sha-256
host  all  postgres  192.168.1.3/32  scram-sha-256

# No wildcard rules — ever
```

### Network Security

| Aspect | Our Lab | Stage | Production Standard | Why |
|--------|---------|-------|---------------------|-----|
| Firewall on DB nodes | Configured | Unknown | ✅ Required — only necessary ports | Even on internal networks, firewall limits blast radius if another server is compromised. |
| PostgreSQL directly reachable | Bound to specific IP | Specific IP ✅ | ✅ Never `0.0.0.0` | Binding to `0.0.0.0` exposes PostgreSQL on all interfaces including any public-facing one. Specific IP binding limits exposure. |
| etcd ports public | Unknown | Unknown | ❌ Only between cluster nodes | etcd holds the cluster's source of truth. External access allows reading or manipulating who the primary is. |
| Patroni REST API | Open to all | `0.0.0.0:8008` | Restricted to HAProxy and monitoring IPs | REST API exposes cluster state and allows operations like switchover if not protected. Should be reachable only by HAProxy and monitoring. |
| TLS for etcd | ❌ | ❌ | ✅ | Unencrypted etcd traffic can be intercepted and manipulated on the wire — complete cluster takeover possible. |
| TLS for PostgreSQL | ❌ | ❌ | ✅ if clients outside internal network | Without TLS, query results and passwords travel in plaintext — interception risk on any shared network. |

### Password and Secrets Management

| Aspect | Our Lab | Stage | Production Standard | Why |
|--------|---------|-------|---------------------|-----|
| Passwords in config files | Plaintext | Plaintext | **Vault, env vars, or encrypted** | Config files are often committed to Git, shared via Ansible, or readable by multiple users. Plaintext passwords in these files are a routine source of credential leaks. |
| Superuser password strength | Weak | Strong | ✅ Strong random | Weak passwords are trivially brute-forced. A compromised superuser means complete database access — all data, all users. |
| Replication password | Medium | 🚨 Same as username | ✅ Strong random | Replication user has `REPLICATION` privilege — it can stream all data off the primary. A weak password here is a data exfiltration risk. |
| Rotation policy | None | None | **Quarterly rotation** | Credentials that never change give an attacker unlimited time to exploit a leaked password. Rotation limits the window of exposure. |

**Pros/Cons:**

| | Our Lab | Stage | Production Standard |
|--|---------|-------|---------------------|
| No wildcard pg_hba rules | ✅ | 🚨 Has `0.0.0.0/0` | ✅ |
| Auth method | ⚠️ md5 | ⚠️ md5 | ✅ scram-sha-256 |
| Secrets management | ❌ Plaintext | ❌ Plaintext | ✅ Vault/encrypted |
| etcd TLS | ❌ | ❌ | ✅ |
| Replication password | ⚠️ Medium | 🚨 Weak | ✅ Strong |

---

## 10. Monitoring and Alerting

Neither our lab setup nor the stage server has monitoring configured. This is one of the most important gaps between a working setup and a production-ready one.

### What to Monitor

**Patroni:**
```bash
# Cluster health — run every minute via cron or monitoring agent
patronictl -c /etc/patroni/patroni.yml list

# REST API response — HAProxy does this, but monitoring should too
curl -s http://node1:8008/patroni | jq '.state, .role'
```

**PostgreSQL:**
```sql
-- Replication lag in bytes (run on primary)
SELECT application_name,
       pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes,
       sync_state
FROM pg_stat_replication;

-- Replication slot health (critical — stale slots accumulate WAL)
SELECT slot_name, active,
       pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) AS lag_bytes
FROM pg_replication_slots;

-- Long running queries (potential lock issues)
SELECT pid, now() - query_start AS duration, query
FROM pg_stat_activity
WHERE state = 'active' AND now() - query_start > interval '5 minutes';

-- Bloat (autovacuum health)
SELECT schemaname, tablename, n_dead_tup, last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC LIMIT 10;
```

**etcd:**
```bash
# DB size vs quota
etcdctl endpoint status --write-out=json | jq '.[].Status.dbSize'

# Cluster health
etcdctl endpoint health --cluster
```

**System:**
```bash
# Disk usage on data directories
df -h /data/pgsql /data/etcd

# pg_wal size
du -sh /data/pgsql/pg_wal

# Archive directory growth
du -sh /log/archive/
```

### Alert Thresholds

| Metric | Warning | Critical |
|--------|---------|----------|
| Replication lag | > 10MB | > 100MB |
| Replication slot lag | > 1GB | > 5GB |
| pg_wal directory size | > 5GB | > 10GB |
| etcd DB size | > 4GB | > 6GB |
| Data disk usage | > 70% | > 85% |
| No primary in cluster | Immediate | Immediate |
| Archive gap | > 1 hour | > 3 hours |
| Query duration | > 5 min | > 30 min |

### Recommended Tools

| Tool | Purpose |
|------|---------|
| **Prometheus + postgres_exporter** | PostgreSQL metrics collection |
| **Prometheus + patroni_exporter** | Patroni cluster metrics |
| **Grafana** | Dashboard visualization |
| **Alertmanager** | Alert routing (email, Slack, PagerDuty) |
| **pgBadger** | PostgreSQL log analysis |
| **check_patroni** | Nagios/Icinga compatible Patroni health checks |

---

## 11. Summary Scorecard

| Category | Our Lab | Stage Server | Production Standard |
|----------|---------|--------------|---------------------|
| **Component versions** | ✅ PG16 current, etcd latest | ⚠️ etcd outdated | ✅ All latest |
| **Directory structure** | ⚠️ Not on dedicated disks | ⚠️ Partial separation | ✅ Full separation |
| **etcd config** | ✅ File based, quota set | ⚠️ Command line, quota unknown | ✅ File, quota, TLS |
| **PG memory tuning** | ❌ Defaults | ✅ Tuned | ✅ Tuned for workload |
| **Replication type** | ❌ Async | ✅ Quorum sync | ✅ Quorum sync |
| **WAL archiving** | ❌ Not configured | ⚠️ Local only | ✅ Remote, monitored |
| **PITR capability** | ❌ | ⚠️ Possible if archive works | ✅ Tested and verified |
| **pg_hba security** | ✅ No wildcards | 🚨 Open wildcards | ✅ Strict per-IP |
| **Auth method** | ⚠️ md5 | ⚠️ md5 | ✅ scram-sha-256 |
| **Password strength** | ⚠️ Weak | ⚠️ Mixed | ✅ All strong |
| **Secrets management** | ❌ Plaintext | ❌ Plaintext | ✅ Vault/encrypted |
| **etcd TLS** | ❌ | ❌ | ✅ |
| **HAProxy redundancy** | ❌ Single | ❌ Single | ✅ Dual + Keepalived |
| **HAProxy health endpoint** | ✅ `/primary` | ⚠️ `/master` deprecated | ✅ `/primary` |
| **Patroni logs to file** | ❌ | ✅ | ✅ |
| **Monitoring** | ❌ | ❌ Not observed | ✅ Full stack |
| **Backup testing** | ❌ | ❌ Not observed | ✅ Monthly |
| **`max_slot_wal_keep_size`** | ❌ | ❌ | ✅ |
| **Overall production readiness** | 🔴 Lab only | 🟡 Functional, gaps exist | 🟢 Production ready |

---

*Last updated: March 2026. Based on PostgreSQL 17, Patroni 4.1, etcd 3.5.17, HAProxy 2.8.*
