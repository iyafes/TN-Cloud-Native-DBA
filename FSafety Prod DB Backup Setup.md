# Remaining Setup Tasks — New DB Server & Backup Server
## Environment: Rocky Linux 9 | PostgreSQL 17 | MongoDB 8.0 | pgBackRest 2.58

---

## Overview

This document covers all setup tasks performed **after** PostgreSQL 17 and MongoDB 8.0 installation on the new DB server. It picks up from where the installation docs left off and covers:

- pgBackRest installation and configuration
- SSH key exchange between DB server and backup server
- MongoDB backup script setup
- Cron job scheduling
- Verification steps

### Server Reference

| Server | IP | OS | Purpose |
|--------|----|----|---------|
| New DB Server | `10.0.130.206` | Rocky Linux 9 | PostgreSQL 17 + MongoDB 8.0 |
| Backup Server | `10.0.137.78` | Rocky Linux 9 | pgBackRest repo + MongoDB dumps |
| Combined Server | `10.0.5.221` | Ubuntu (Docker) | Current prod (being migrated away from) |

---

## Part 1 — pgBackRest Setup

pgBackRest is used for PostgreSQL backups. It supports full, differential, and incremental backups with WAL archiving, compression, and parallel processing.

### 1.1 Installation

Install pgBackRest on **both** the DB server and the backup server. The PGDG repo must be added first (already done during PG17 install on DB server; needs to be done on backup server separately).

**On DB server (`10.0.130.206`):**
```bash
sudo dnf install -y pgbackrest
pgbackrest version
```

**On backup server (`10.0.137.78`):**
```bash
sudo dnf install -y \
  https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo dnf install -y pgbackrest
pgbackrest version
```

> **Note:** pgBackRest installation via PGDG repo does NOT automatically create a `pgbackrest` OS user on Rocky Linux. You must create it manually (see Section 1.3).

---

### 1.2 Repository Directory Setup

pgBackRest stores backups in a repository. This must be on the **backup server** under `/data` (dedicated NVMe disk — 98GB available).

**On backup server (`10.0.137.78`):**
```bash
sudo mkdir -p /data/pgbackrest
sudo chown pgbackrest:pgbackrest /data/pgbackrest
sudo chmod 750 /data/pgbackrest
```

**SELinux context fix (critical on Rocky Linux):**

The `/data` mount point has `unlabeled_t` context by default — pgBackRest cannot write to it. Fix in two steps:

Step 1 — Fix parent directory first:
```bash
sudo semanage fcontext -a -t var_t "/data(/.*)?"
sudo restorecon -Rv /data
```

Step 2 — Apply specific context for pgBackRest:
```bash
sudo semanage fcontext -a -t var_lib_t "/data/pgbackrest(/.*)?"
sudo restorecon -Rv /data/pgbackrest
```

Verify:
```bash
ls -laZ /data/
# pgbackrest directory should show: unconfined_u:object_r:var_lib_t:s0
```

> **Why two steps?** If the parent directory has `unlabeled_t`, SELinux ignores subdirectory context rules. The parent must be labeled first before subdirectory rules take effect.

---

### 1.3 pgbackrest OS User Creation

On Rocky Linux, the `pgbackrest` package does not create the OS user. Create it manually on the **backup server**:

```bash
sudo useradd --system --no-create-home --shell /bin/bash pgbackrest
sudo mkdir -p /home/pgbackrest
sudo chown pgbackrest:pgbackrest /home/pgbackrest
id pgbackrest  # verify
```

---

### 1.4 SSH Key Exchange

pgBackRest communicates between DB server and backup server via SSH. Keys must be set up in **both directions**:

- DB server `postgres` user → Backup server `pgbackrest` user
- Backup server `pgbackrest` user → DB server `postgres` user

**Direction 1: DB server → Backup server**

On **DB server** (`10.0.130.206`), generate ED25519 key for `postgres` user:
```bash
sudo mkdir -p /var/lib/pgsql/.ssh
sudo chmod 700 /var/lib/pgsql/.ssh
sudo -u postgres ssh-keygen -t ed25519 -f /var/lib/pgsql/.ssh/id_ed25519 -N '' -C 'postgres@db-server'
sudo chown -R postgres:postgres /var/lib/pgsql/.ssh
```

> **Important:** PostgreSQL's home directory is `/var/lib/pgsql/`, NOT `/home/postgres/`. Always use `/var/lib/pgsql/.ssh/` for the `postgres` user's SSH keys.

Get the public key:
```bash
sudo cat /var/lib/pgsql/.ssh/id_ed25519.pub
```

Copy this output. On **backup server** (`10.0.137.78`), add it to pgbackrest's authorized_keys:
```bash
sudo mkdir -p /home/pgbackrest/.ssh
sudo chmod 700 /home/pgbackrest/.ssh
echo 'PASTE_PUBLIC_KEY_HERE' | sudo tee /home/pgbackrest/.ssh/authorized_keys
sudo chmod 600 /home/pgbackrest/.ssh/authorized_keys
sudo chown -R pgbackrest:pgbackrest /home/pgbackrest/.ssh
```

Test from **DB server**:
```bash
sudo -u postgres ssh pgbackrest@10.0.137.78 "echo SSH connection successful"
```

**Direction 2: Backup server → DB server**

On **backup server** (`10.0.137.78`), generate ED25519 key for `pgbackrest` user:
```bash
sudo -u pgbackrest ssh-keygen -t ed25519 -f /home/pgbackrest/.ssh/id_ed25519 -N '' -C 'pgbackrest@backup-server'
```

Get the public key:
```bash
sudo cat /home/pgbackrest/.ssh/id_ed25519.pub
```

On **DB server** (`10.0.130.206`), add to postgres's authorized_keys:
```bash
echo 'PASTE_PUBLIC_KEY_HERE' | sudo tee -a /var/lib/pgsql/.ssh/authorized_keys
sudo chmod 600 /var/lib/pgsql/.ssh/authorized_keys
sudo chown -R postgres:postgres /var/lib/pgsql/.ssh
```

**SELinux fix for postgres .ssh directory (critical):**

The `/var/lib/pgsql/.ssh/` directory inherits `postgresql_db_t` context — sshd cannot read `authorized_keys` with this context. Fix it:

```bash
sudo semanage fcontext -a -t ssh_home_t "/var/lib/pgsql/.ssh(/.*)?"
sudo restorecon -Rv /var/lib/pgsql/.ssh
```

Verify:
```bash
sudo ls -laZ /var/lib/pgsql/.ssh/
# All files should show: ssh_home_t
```

Test from **backup server**:
```bash
sudo -u pgbackrest ssh postgres@10.0.130.206 "echo SSH connection successful"
```

> **Troubleshooting SSH failures:** If connection is refused despite correct keys, always check SELinux first: `sudo ausearch -m avc -ts recent | grep ssh`. The most common cause on Rocky Linux is wrong SELinux context on `.ssh` directories.

---

### 1.5 pgBackRest Configuration

**On DB server** (`10.0.130.206`), create `/etc/pgbackrest/pgbackrest.conf`:
```bash
sudo mkdir -p /etc/pgbackrest
sudo tee /etc/pgbackrest/pgbackrest.conf << 'EOF'
[global]
repo1-host=10.0.137.78
repo1-host-user=pgbackrest
repo1-retention-full=30
repo1-retention-diff=60
log-level-console=info
log-level-file=detail
log-path=/logs/postgresql/pgbackrest

[db1]
pg1-path=/data/pgsql/17/data
pg1-user=postgres
EOF
```

**On backup server** (`10.0.137.78`), create `/etc/pgbackrest/pgbackrest.conf`:
```bash
sudo mkdir -p /etc/pgbackrest
sudo tee /etc/pgbackrest/pgbackrest.conf << 'EOF'
[global]
repo1-path=/data/pgbackrest
repo1-retention-full=30
repo1-retention-diff=60
log-level-console=info
log-level-file=detail
log-path=/var/log/pgbackrest

[db1]
pg1-host=10.0.130.206
pg1-host-user=postgres
pg1-path=/data/pgsql/17/data
EOF
```

**Retention explanation:**
- `repo1-retention-full=30` — keep 30 full backups (30 days worth since we take 1 full/day)
- `repo1-retention-diff=60` — keep 60 differential backups (2 per day × 30 days)

---

### 1.6 Log Directory Setup

**On DB server** (`10.0.130.206`):
```bash
sudo mkdir -p /logs/postgresql/pgbackrest
sudo chown postgres:postgres /logs/postgresql/pgbackrest
```

**On backup server** (`10.0.137.78`):
```bash
sudo mkdir -p /var/log/pgbackrest
sudo chown pgbackrest:pgbackrest /var/log/pgbackrest
```

---

### 1.7 Update archive_command in PostgreSQL

pgBackRest requires its own `archive_command`. Update it on **DB server**:

```bash
sudo -u postgres psql -c "ALTER SYSTEM SET archive_command = 'pgbackrest --stanza=db1 archive-push %p';"
sudo -u postgres psql -c "SELECT pg_reload_conf();"
sudo -u postgres psql -c "SHOW archive_command;"
```

> **Why?** The default `archive_command` used `cp` to copy WAL files locally. pgBackRest needs to receive WAL segments itself to maintain a consistent archive on the backup server. Without this, `pgbackrest check` will fail with error [068].

---

### 1.8 Stanza Create

A stanza is pgBackRest's configuration unit for a PostgreSQL cluster. Run from **DB server**:

```bash
sudo -u postgres pgbackrest --stanza=db1 --log-level-console=info stanza-create
```

Expected output:
```
INFO: stanza-create command end: completed successfully
```

---

### 1.9 Verify Configuration

Run from **DB server**:
```bash
sudo -u postgres pgbackrest --stanza=db1 --log-level-console=info check
```

Expected output includes:
```
INFO: WAL segment XXXXXXXXXXXXXXXXXXXXXXXXXX successfully archived to '/data/pgbackrest/archive/...'
INFO: check command end: completed successfully
```

> **If check fails with archive_command error:** The `archive_command` in `postgresql.conf` must contain `pgbackrest`. Run the ALTER SYSTEM command in Section 1.7.

---

### 1.10 Take First Full Backup

From **DB server**:
```bash
sudo -u postgres pgbackrest --stanza=db1 --log-level-console=info backup --type=full
```

Verify backup exists on **backup server**:
```bash
sudo find /data/pgbackrest/backup -type d | head -10
```

---

### 1.11 Cron Job Setup

Schedule automated backups as `postgres` user on **DB server**:

```bash
sudo -u postgres crontab -e
```

Add:
```
0 5  * * * pgbackrest --stanza=db1 --log-level-file=detail backup --type=full
0 13 * * * pgbackrest --stanza=db1 --log-level-file=detail backup --type=diff
0 20 * * * pgbackrest --stanza=db1 --log-level-file=detail backup --type=diff
```

Verify:
```bash
sudo -u postgres crontab -l
```

**Schedule explanation (server runs UTC):**
- `0 5` UTC = 11:00 AM local (+06) — full backup
- `0 13` UTC = 7:00 PM local — differential backup  
- `0 20` UTC = 2:00 AM local — differential backup

> **Note:** All times are UTC on the server. Adjust accordingly for your timezone.

---

## Part 2 — MongoDB Backup Setup

MongoDB does not have a tool like pgBackRest. The backup strategy uses `mongodump` + `rsync` via cron.

### 2.1 SSH Key Exchange for MongoDB Backup

MongoDB backup runs as the `mongod` OS user. A separate SSH key is needed for transferring dumps to the backup server.

> **Important:** MongoDB's home directory is `/var/lib/mongo/`, NOT `/home/mongod/`. Always use `/var/lib/mongo/.ssh/` for the `mongod` user's SSH keys.

**On DB server** (`10.0.130.206`), generate key for `mongod` user:
```bash
sudo mkdir -p /var/lib/mongo/.ssh
sudo chmod 700 /var/lib/mongo/.ssh
sudo chown mongod:mongod /var/lib/mongo/.ssh
sudo -u mongod ssh-keygen -t ed25519 -f /var/lib/mongo/.ssh/id_ed25519 -N '' -C 'mongod@db-server'
```

**SELinux fix for mongod .ssh directory:**
```bash
sudo semanage fcontext -a -t ssh_home_t "/var/lib/mongo/.ssh(/.*)?"
sudo restorecon -Rv /var/lib/mongo/.ssh
```

Get the public key:
```bash
sudo cat /var/lib/mongo/.ssh/id_ed25519.pub
```

**On backup server** (`10.0.137.78`), create `mongodbackup` user and add key:
```bash
sudo useradd --system --no-create-home --shell /bin/bash mongodbackup
sudo mkdir -p /home/mongodbackup/.ssh
sudo chown mongodbackup:mongodbackup /home/mongodbackup
sudo chmod 700 /home/mongodbackup/.ssh
echo 'PASTE_PUBLIC_KEY_HERE' | sudo tee /home/mongodbackup/.ssh/authorized_keys
sudo chmod 600 /home/mongodbackup/.ssh/authorized_keys
sudo chown -R mongodbackup:mongodbackup /home/mongodbackup/.ssh
```

Test from **DB server**:
```bash
sudo -u mongod ssh mongodbackup@10.0.137.78 "echo SSH connection successful"
```

---

### 2.2 Backup Directory Setup

**On backup server** (`10.0.137.78`):
```bash
sudo mkdir -p /data/mongodb-backup
sudo chown mongodbackup:mongodbackup /data/mongodb-backup
sudo chmod 750 /data/mongodb-backup
```

---

### 2.3 Backup Script

Create the backup script on **DB server** (`10.0.130.206`):

```bash
sudo tee /usr/local/bin/mongodb-backup.sh << 'EOF'
#!/bin/bash

# MongoDB backup script
# Runs mongodump, transfers to backup server, cleans up old backups

# Variables
DATE=$(date +%Y-%m-%d_%H-%M-%S)
BACKUP_DIR="/var/lib/mongo/backup"
DUMP_DIR="$BACKUP_DIR/mongodump-$DATE"
BACKUP_SERVER="mongodbackup@10.0.137.78"
BACKUP_SERVER_DIR="/data/mongodb-backup"
RETENTION_DAYS=30
LOG_FILE="/logs/mongodb/backup.log"

echo "[$DATE] MongoDB backup started" >> $LOG_FILE

mkdir -p $DUMP_DIR

# Dump all databases
mongodump \
  --host 127.0.0.1 \
  --port 27017 \
  --username FSafetyProd \
  --password FSh7h9w4MDBProD \
  --authenticationDatabase admin \
  --out $DUMP_DIR >> $LOG_FILE 2>&1

if [ $? -ne 0 ]; then
  echo "[$DATE] mongodump FAILED" >> $LOG_FILE
  exit 1
fi

echo "[$DATE] mongodump successful" >> $LOG_FILE

# Transfer to backup server
rsync -avz \
  -e "ssh -i /var/lib/mongo/.ssh/id_ed25519" \
  $DUMP_DIR \
  $BACKUP_SERVER:$BACKUP_SERVER_DIR/ >> $LOG_FILE 2>&1

if [ $? -ne 0 ]; then
  echo "[$DATE] rsync FAILED" >> $LOG_FILE
  exit 1
fi

echo "[$DATE] rsync successful" >> $LOG_FILE

# Remove local backups older than RETENTION_DAYS
find $BACKUP_DIR -maxdepth 1 -name "mongodump-*" -mtime +$RETENTION_DAYS -exec rm -rf {} \;

echo "[$DATE] MongoDB backup completed" >> $LOG_FILE
EOF
```

Set permissions:
```bash
sudo chmod +x /usr/local/bin/mongodb-backup.sh
sudo chown mongod:mongod /usr/local/bin/mongodb-backup.sh
```

> **Credentials note:** The script uses `FSafetyProd` — the only MongoDB user that exists on the combined server. After real migration, this user's credentials will carry over via mongorestore. If credentials change, update the script accordingly.

---

### 2.4 Test the Backup Script

```bash
sudo -u mongod /usr/local/bin/mongodb-backup.sh
sudo tail -20 /logs/mongodb/backup.log
```

Expected log output ends with:
```
[TIMESTAMP] rsync successful
[TIMESTAMP] MongoDB backup completed
```

> **Note:** `find: Failed to restore initial working directory: /home/rocky: Permission denied` — this warning is harmless. The `mongod` user cannot access `/home/rocky` but the backup completes successfully.

Verify on **backup server**:
```bash
sudo ls -lh /data/mongodb-backup/
```

---

### 2.5 Cron Job Setup

Schedule as `mongod` user on **DB server**:

```bash
sudo -u mongod crontab -e
```

Add:
```
0 5  * * * /usr/local/bin/mongodb-backup.sh
0 13 * * * /usr/local/bin/mongodb-backup.sh
0 20 * * * /usr/local/bin/mongodb-backup.sh
```

Verify:
```bash
sudo -u mongod crontab -l
```

---

## Part 3 — pg_hba.conf — Backup Server Access

Add backup server IP to allow pgBackRest replication connections. On **DB server** (`10.0.130.206`):

```bash
sudo tee -a /data/pgsql/17/data/pg_hba.conf << 'EOF'

# Backup server
host    all             all             10.0.137.78/32          scram-sha-256
EOF

sudo -u postgres psql -c "SELECT pg_reload_conf();"
sudo grep 10.0.137.78 /data/pgsql/17/data/pg_hba.conf
```

---

## Part 4 — Final Verification Checklist

### PostgreSQL Backup
```bash
# On DB server
sudo -u postgres pgbackrest --stanza=db1 --log-level-console=info check
sudo -u postgres pgbackrest --stanza=db1 info
sudo -u postgres crontab -l
sudo tail -20 /logs/postgresql/pgbackrest/db1-backup.log
```

### MongoDB Backup
```bash
# On DB server
sudo -u mongod crontab -l
sudo tail -20 /logs/mongodb/backup.log

# On backup server
sudo ls -lh /data/mongodb-backup/
sudo ls -lh /data/pgbackrest/backup/db1/
```

### Services
```bash
# On DB server
sudo systemctl status postgresql-17
sudo systemctl status mongod
```

---

## Troubleshooting Reference

| Issue | Symptom | Fix |
|-------|---------|-----|
| pgBackRest stanza-create fails | `requires option: pg1-path` | Config file missing — create `/etc/pgbackrest/pgbackrest.conf` |
| SSH permission denied | `Authentication failed (publickey)` | Check SELinux context with `ausearch -m avc -ts recent \| grep ssh` |
| postgres .ssh wrong context | `avc: denied { read } for authorized_keys` | `semanage fcontext -a -t ssh_home_t "/var/lib/pgsql/.ssh(/.*)?"; restorecon -Rv` |
| pgBackRest check fails | Error [068] archive_command must contain pgbackrest | Run `ALTER SYSTEM SET archive_command = 'pgbackrest --stanza=db1 archive-push %p'` |
| MongoDB SSH key not found | `identity file /var/lib/mongo/.ssh/... type -1` | Key is in `/home/mongod/.ssh/` — move to `/var/lib/mongo/.ssh/` |
| mongod user home directory | Keys not found | mongod's home is `/var/lib/mongo/`, not `/home/mongod/` |
| pgbackrest user missing | `chown: invalid user: pgbackrest:pgbackrest` | `useradd --system --no-create-home --shell /bin/bash pgbackrest` |
| Backup deleted, stanza missing | `status: error (missing stanza path)` | Run `pgbackrest --stanza=db1 stanza-create` again |
