# MySQL 8.4 — Complete Setup & Streaming Replication Guide

> **Environment:** Rocky Linux 9 | MySQL 8.4 Community Edition  
> **Master Server IP:** `172.16.93.140`  
> **Replica Server IP:** `172.16.93.141`

---

## Table of Contents

1. [Overview — What is Streaming Replication?](#1-overview--what-is-streaming-replication)
2. [Architecture](#2-architecture)
3. [Part 1 — MySQL Installation (Both Servers)](#3-part-1--mysql-installation-both-servers)
4. [Part 2 — Secure Installation (Both Servers)](#4-part-2--secure-installation-both-servers)
5. [Part 3 — Change Data Directory (Both Servers)](#5-part-3--change-data-directory-both-servers)
6. [Part 4 — Configure Master Server](#6-part-4--configure-master-server)
7. [Part 5 — Configure Replica Server](#7-part-5--configure-replica-server)
8. [Part 6 — Start & Verify Replication](#8-part-6--start--verify-replication)
9. [Part 7 — Live Test](#9-part-7--live-test)
10. [Troubleshooting](#10-troubleshooting)
11. [Quick Reference](#11-quick-reference)

---

## 1. Overview — What is Streaming Replication?

MySQL Streaming Replication is a mechanism where every data change that happens on a **Master (Source)** server is automatically and continuously replicated to one or more **Replica (Slave)** servers in near real-time.

### How it works — step by step

```
Master Server                          Replica Server
─────────────────                      ─────────────────
User writes data                       
    │                                  
    ▼                                  
Binary Log (binlog)  ──────────────►  I/O Thread reads binlog
    │                                       │
    │                                       ▼
    │                                  Relay Log (local copy)
    │                                       │
    │                                       ▼
    │                                  SQL Thread applies changes
    │                                       │
    │                                       ▼
    │                                  Replica Database (in sync)
```

1. Every `INSERT`, `UPDATE`, `DELETE` on Master is written to the **Binary Log (binlog)**
2. Replica's **I/O Thread** connects to Master and continuously reads the binlog
3. The binlog events are stored locally in the Replica's **Relay Log**
4. Replica's **SQL Thread** reads the Relay Log and applies the changes to its own database
5. Both servers stay in sync

### Why use replication?

| Use Case | Description |
|---|---|
| **High Availability** | If Master goes down, Replica can be promoted |
| **Read Scaling** | Distribute read queries across Replica servers |
| **Backup** | Take backups from Replica without impacting Master |
| **Disaster Recovery** | Replica in a different location acts as DR site |

---

## 2. Architecture

```
┌─────────────────────────────────┐        ┌─────────────────────────────────┐
│         MASTER SERVER           │        │         REPLICA SERVER          │
│       172.16.93.140             │        │       172.16.93.141             │
│                                 │        │                                 │
│  MySQL 8.4                      │        │  MySQL 8.4                      │
│  server-id = 1                  │        │  server-id = 2                  │
│  log_bin   = ON                 │        │  read_only = ON                 │
│  datadir   = /data/mysql        │        │  datadir   = /data/mysql        │
│                                 │        │                                 │
│  Binlog ──────────────────────────────►  Relay Log                        │
└─────────────────────────────────┘        └─────────────────────────────────┘
              Port 3306                              Port 3306
```

> **Note:** Both servers follow the identical initial setup process (install → secure → data directory change). Only the replication-specific configuration differs.

---

## 3. Part 1 — MySQL Installation (Both Servers)

> Perform these steps on **both** the Master (`172.16.93.140`) and Replica (`172.16.93.141`) servers.

### Step 1.1 — Update the system

```bash
sudo dnf update -y
```

Always update the system before installing new software to ensure you have the latest security patches and compatible package versions.

### Step 1.2 — Add the MySQL 8.4 repository

```bash
sudo dnf install -y https://dev.mysql.com/get/mysql84-community-release-el9-1.noarch.rpm
```

Rocky Linux does not include MySQL in its default repositories. This command downloads and installs the official MySQL repository configuration from MySQL's website, which allows `dnf` to find and install MySQL 8.4 packages.

> **For Rocky Linux 8**, replace `el9` with `el8` in the URL.

### Step 1.3 — Install MySQL Server

```bash
sudo dnf install -y mysql-community-server
```

This installs the MySQL Community Server package along with all required dependencies.

### Step 1.4 — Start and enable MySQL service

```bash
sudo systemctl start mysqld
sudo systemctl enable mysqld
```

- `start` — starts MySQL immediately
- `enable` — ensures MySQL starts automatically every time the server reboots

### Step 1.5 — Verify MySQL is running

```bash
sudo systemctl status mysqld
```

You should see `Active: active (running)` in the output.

---

## 4. Part 2 — Secure Installation (Both Servers)

> Perform these steps on **both** servers.

### Step 2.1 — Retrieve the temporary root password

When MySQL is installed for the first time, it generates a temporary random password for the `root` user and writes it to the error log.

```bash
sudo grep 'temporary password' /var/log/mysqld.log
```

**Example output:**
```
A temporary password is generated for root@localhost: AbCdEf@12345
```

Copy this password — you will need it in the next step.

### Step 2.2 — Run the secure installation wizard

```bash
sudo mysql_secure_installation
```

The wizard will prompt you through the following steps in order:

| Prompt | Action |
|---|---|
| `Enter password for user root` | Enter the temporary password from Step 2.1 |
| `Change the password for root?` | Press `Y` |
| `New password` | Enter a strong password (uppercase + lowercase + number + special character) |
| `Re-enter new password` | Confirm the new password |
| `Do you wish to continue with the password provided?` | Press `Y` |
| `Remove anonymous users?` | Press `Y` |
| `Disallow root login remotely?` | Press `Y` |
| `Remove test database?` | Press `Y` |
| `Reload privilege tables?` | Press `Y` |

> **Password Policy:** MySQL 8.4 enforces a strong password policy by default. Your password must include at least one uppercase letter, one lowercase letter, one digit, and one special character (e.g., `MyPass@2024!`).

### Step 2.3 — Verify login with the new password

```bash
mysql -u root -p
```

Enter your new password. If you see the `mysql>` prompt, login is successful. Type `exit` to leave.

---

## 5. Part 3 — Change Data Directory (Both Servers)

> The default MySQL data directory is `/var/lib/mysql`. This section moves it to `/data/mysql` — a common requirement when you want data on a separate volume or disk. Perform these steps on **both** servers.

### Step 3.1 — Stop MySQL service

```bash
sudo systemctl stop mysqld
```

MySQL must be stopped before moving its data files to prevent corruption.

### Step 3.2 — Create the new data directory

```bash
sudo mkdir -p /data/mysql
```

The `-p` flag creates parent directories as needed. If `/data` does not exist, it will be created automatically.

### Step 3.3 — Copy existing data to the new directory

```bash
sudo rsync -av /var/lib/mysql/ /data/mysql/
```

> If `rsync` is not installed: `sudo dnf install -y rsync`

`rsync` is preferred over `cp` because it preserves file permissions, ownership, timestamps, and symbolic links — all of which are critical for MySQL to function correctly.

- `-a` (archive) — preserves permissions, ownership, symlinks, and timestamps
- `-v` (verbose) — shows each file being copied

### Step 3.4 — Set correct ownership

```bash
sudo chown -R mysql:mysql /data/mysql
sudo chmod 750 /data/mysql
```

MySQL runs as the `mysql` system user. The data directory and all its contents must be owned by this user, otherwise MySQL will refuse to start.

### Step 3.5 — Configure SELinux context

This is a **critical step** specific to Rocky Linux / RHEL-based systems.

```bash
sudo semanage fcontext -a -t mysqld_db_t "/data/mysql(/.*)?"
sudo restorecon -Rv /data/mysql
```

> If `semanage` is not found: `sudo dnf install -y policycoreutils-python-utils`

**Why this is required:**  
SELinux (Security-Enhanced Linux) is enabled by default on Rocky Linux. It assigns a security label (called a "context") to every file and directory. The default label on `/data/mysql` would be `default_t`, which SELinux does not allow MySQL to access.

- `semanage fcontext` — registers a new rule that says "files under `/data/mysql` should have the label `mysqld_db_t`"
- `restorecon` — physically applies that label to all existing files and directories
- Without this step, MySQL will fail to start even if permissions are correct

### Step 3.6 — Update MySQL configuration file

```bash
sudo vi /etc/my.cnf
```

Update the `[mysqld]` and `[client]` sections so MySQL knows where to find its data:

```ini
[mysqld]
datadir=/data/mysql
socket=/data/mysql/mysql.sock
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid

[client]
socket=/data/mysql/mysql.sock
```

- `datadir` — tells MySQL where to store databases and tables
- `socket` — the Unix socket file used for local connections; must match between `[mysqld]` and `[client]`

### Step 3.7 — Start MySQL and verify

```bash
sudo systemctl start mysqld
sudo systemctl status mysqld
```

Confirm the new data directory is active:

```bash
mysql -u root -p -e "SHOW VARIABLES LIKE 'datadir';"
```

**Expected output:**
```
+---------------+-------------+
| Variable_name | Value       |
+---------------+-------------+
| datadir       | /data/mysql/|
+---------------+-------------+
```

---

## 6. Part 4 — Configure Master Server

> Perform these steps on the **Master server only** (`172.16.93.140`).

### Step 4.1 — Edit the MySQL configuration file

```bash
sudo vi /etc/my.cnf
```

Add the following lines inside the `[mysqld]` section:

```ini
[mysqld]
datadir=/data/mysql
socket=/data/mysql/mysql.sock
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid

# --- Replication Settings ---
server-id                  = 1
log_bin                    = /data/mysql/mysql-bin.log
binlog_format              = ROW
binlog_expire_logs_seconds = 604800
```

**Explanation of each replication parameter:**

| Parameter | Value | Purpose |
|---|---|---|
| `server-id` | `1` | A unique numeric ID for this server. Every MySQL server in a replication setup must have a different ID. Master = 1, Replica = 2. |
| `log_bin` | `/data/mysql/mysql-bin.log` | Enables the Binary Log and sets its file path. This is the core of replication — every data change is recorded here. Without this, replication cannot work. |
| `binlog_format` | `ROW` | Determines what is written to the binlog. `ROW` format logs the actual before/after state of each row changed — more reliable than `STATEMENT` mode, especially for stored procedures and non-deterministic functions. |
| `binlog_expire_logs_seconds` | `604800` | Automatically deletes binlog files older than 7 days (604800 seconds) to prevent the disk from filling up. |

### Step 4.2 — Restart MySQL to apply changes

```bash
sudo systemctl restart mysqld
```

### Step 4.3 — Verify Binary Logging is enabled

```bash
mysql -u root -p -e "SHOW VARIABLES LIKE 'log_bin';"
```

**Expected output:**
```
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| log_bin       | ON    |
+---------------+-------+
```

### Step 4.4 — Create a dedicated replication user

```bash
mysql -u root -p
```

```sql
-- Create a user restricted to the Replica's IP address
CREATE USER 'replica'@'172.16.93.141' IDENTIFIED WITH caching_sha2_password BY 'Replica@123';

-- Grant only the replication privilege — nothing else
GRANT REPLICATION SLAVE ON *.* TO 'replica'@'172.16.93.141';

-- Apply the privilege changes immediately
FLUSH PRIVILEGES;
```

**Why a dedicated user?**  
It is a security best practice to create a separate user exclusively for replication with the minimum required privilege (`REPLICATION SLAVE`). This user cannot read or write any data — it can only read the binlog stream.

**Why `@'172.16.93.141'`?**  
Unlike PostgreSQL which has a separate `pg_hba.conf` file for IP-based access control, MySQL handles host restriction directly in the user definition using the `@'host'` syntax. This means the `replica` user can only connect from the Replica server's IP — no other machine can use this account.

**Why `caching_sha2_password`?**  
MySQL 8.4 removed `mysql_native_password` entirely. The only supported authentication plugin is `caching_sha2_password`. When used without SSL, the Replica must request the Master's public key to authenticate — this is handled by `GET_SOURCE_PUBLIC_KEY = 1` in the Replica configuration (covered in Part 5).

### Step 4.5 — Verify the user was created

```sql
SELECT user, host, plugin FROM mysql.user WHERE user = 'replica';
```

**Expected output:**
```
+---------+----------------+------------------------+
| user    | host           | plugin                 |
+---------+----------------+------------------------+
| replica | 172.16.93.141  | caching_sha2_password  |
+---------+----------------+------------------------+
```

### Step 4.6 — Record the current binlog position

```sql
SHOW BINARY LOG STATUS\G
```

> **Note:** `SHOW MASTER STATUS` is deprecated in MySQL 8.4. Use `SHOW BINARY LOG STATUS` instead.

**Example output:**
```
*************************** 1. row ***************************
             File: mysql-bin.000001
         Position: 887
     Binlog_Do_DB:
 Binlog_Ignore_DB:
```

**Record these two values immediately:**
- `File: mysql-bin.000001`
- `Position: 887`

These tell the Replica exactly where to start reading the binlog. The Position advances with every write operation, so record these values and configure the Replica promptly. Avoid running any write operations on the Master between this step and completing the Replica setup.

### Step 4.7 — Open firewall for the Replica

```bash
sudo firewall-cmd --permanent --add-rich-rule='rule family=ipv4 source address=172.16.93.141 port port=3306 protocol=tcp accept'
sudo firewall-cmd --reload
```

This allows only the Replica server (`172.16.93.141`) to connect to MySQL port 3306 on the Master. Other machines remain blocked.

---

## 7. Part 5 — Configure Replica Server

> Perform these steps on the **Replica server only** (`172.16.93.141`).

### Step 5.1 — Edit the MySQL configuration file

```bash
sudo vi /etc/my.cnf
```

```ini
[mysqld]
datadir=/data/mysql
socket=/data/mysql/mysql.sock
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid

# --- Replication Settings ---
server-id          = 2
read_only          = ON
relay_log          = /data/mysql/relay-bin.log
relay_log_recovery = ON
```

**Explanation of each replication parameter:**

| Parameter | Value | Purpose |
|---|---|---|
| `server-id` | `2` | Must be different from the Master's `server-id = 1`. Each server in the replication topology must have a unique ID. |
| `read_only` | `ON` | Prevents any application or user (except replication threads and root) from writing directly to the Replica. This ensures data on the Replica only comes from the Master, keeping both servers in sync. |
| `relay_log` | `/data/mysql/relay-bin.log` | The Relay Log is a local copy of the Master's binlog. The I/O Thread writes to it, and the SQL Thread reads from it. Setting the path keeps it with the data directory. |
| `relay_log_recovery` | `ON` | If the Replica crashes, this setting discards any incomplete relay log files and re-fetches them from the Master on restart. This prevents data corruption from partially written relay logs. |

### Step 5.2 — Restart MySQL to apply changes

```bash
sudo systemctl restart mysqld
```

### Step 5.3 — Connect Replica to Master

```bash
mysql -u root -p
```

```sql
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST           = '172.16.93.140',
  SOURCE_USER           = 'replica',
  SOURCE_PASSWORD       = 'Replica@123',
  SOURCE_LOG_FILE       = 'mysql-bin.000001',
  SOURCE_LOG_POS        = 887,
  GET_SOURCE_PUBLIC_KEY = 1;
```

**Explanation of each parameter:**

| Parameter | Value | Description |
|---|---|---|
| `SOURCE_HOST` | `172.16.93.140` | IP address of the Master server |
| `SOURCE_USER` | `replica` | The replication user created on the Master in Step 4.4 |
| `SOURCE_PASSWORD` | `Replica@123` | Password for the replication user |
| `SOURCE_LOG_FILE` | `mysql-bin.000001` | The binlog filename recorded in Step 4.6 |
| `SOURCE_LOG_POS` | `887` | The binlog position recorded in Step 4.6 — Replica will start reading from this exact point |
| `GET_SOURCE_PUBLIC_KEY` | `1` | **Critical for MySQL 8.4.** Since `caching_sha2_password` requires either SSL or a public key exchange, this tells the Replica to automatically fetch the Master's RSA public key for password authentication. Without this, the connection fails with an authentication error. |

---

## 8. Part 6 — Start & Verify Replication

> Perform these steps on the **Replica server**.

### Step 6.1 — Start replication

```sql
START REPLICA;
```

### Step 6.2 — Check replication status

```sql
SHOW REPLICA STATUS\G
```

> **Note:** `SHOW SLAVE STATUS` is deprecated in MySQL 8.4. Use `SHOW REPLICA STATUS` instead.

### Step 6.3 — Interpret the status output

A healthy replication setup shows:

```
Replica_IO_Running: Yes
Replica_SQL_Running: Yes
Seconds_Behind_Source: 0
Last_IO_Error:
Last_SQL_Error:
```

| Field | Expected Value | Meaning |
|---|---|---|
| `Replica_IO_Running` | `Yes` | The I/O Thread is connected to the Master and successfully reading the binlog |
| `Replica_SQL_Running` | `Yes` | The SQL Thread is reading the Relay Log and applying changes to the Replica database |
| `Seconds_Behind_Source` | `0` | The Replica is fully caught up with the Master. A higher number means the Replica is lagging behind. |
| `Last_IO_Error` | *(empty)* | No connection or authentication errors |
| `Last_SQL_Error` | *(empty)* | No errors applying changes from the relay log |

Both `Replica_IO_Running` and `Replica_SQL_Running` must be `Yes` for replication to be working correctly.

---

## 9. Part 7 — Live Test

Confirm that data written on the Master is automatically replicated to the Replica.

### On the Master server (`172.16.93.140`):

```sql
CREATE DATABASE repl_test;
USE repl_test;

CREATE TABLE test_table (
  id         INT AUTO_INCREMENT PRIMARY KEY,
  message    VARCHAR(100),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO test_table (message) VALUES ('Replication is working!');
INSERT INTO test_table (message) VALUES ('Row 2 from Master');
```

### On the Replica server (`172.16.93.141`) — wait a few seconds, then:

```sql
USE repl_test;
SELECT * FROM test_table;
```

**Expected output:**
```
+----+-------------------------+---------------------+
| id | message                 | created_at          |
+----+-------------------------+---------------------+
|  1 | Replication is working! | 2024-03-08 14:00:00 |
|  2 | Row 2 from Master       | 2024-03-08 14:00:01 |
+----+-------------------------+---------------------+
```

If both rows appear on the Replica, **streaming replication is fully operational**.

---

## 10. Troubleshooting

### Issue 1: `Replica_IO_Running: Connecting` or `No`

**Symptom:** The I/O Thread cannot connect to the Master.

**Check the error:**
```sql
SHOW REPLICA STATUS\G
-- Look at: Last_IO_Error
```

| Error Message | Cause | Solution |
|---|---|---|
| `Authentication plugin 'caching_sha2_password' requires secure connection` | `GET_SOURCE_PUBLIC_KEY` not set | Add `GET_SOURCE_PUBLIC_KEY = 1` to `CHANGE REPLICATION SOURCE TO` |
| `Access denied for user 'replica'@...` | Wrong password or user does not exist | Recreate the user on Master with correct credentials |
| `Can't connect to MySQL server on '172.16.93.140'` | Network or firewall issue | Check firewall rules on Master; verify port 3306 is open |

**Reset and reconfigure:**
```sql
STOP REPLICA;
RESET REPLICA ALL;

CHANGE REPLICATION SOURCE TO
  SOURCE_HOST           = '172.16.93.140',
  SOURCE_USER           = 'replica',
  SOURCE_PASSWORD       = 'Replica@123',
  SOURCE_LOG_FILE       = 'mysql-bin.000001',
  SOURCE_LOG_POS        = 887,
  GET_SOURCE_PUBLIC_KEY = 1;

START REPLICA;
```

---

### Issue 2: `Replica_SQL_Running: No`

**Symptom:** The SQL Thread has stopped applying changes.

**Check the error:**
```sql
SHOW REPLICA STATUS\G
-- Look at: Last_SQL_Error and Last_SQL_Errno
```

**Common cause:** The SQL Thread tried to apply a statement from the Master that failed on the Replica (e.g., a user or object already exists or does not exist).

**Skip the failed transaction and continue:**
```sql
STOP REPLICA;
SET GLOBAL SQL_REPLICA_SKIP_COUNTER = 1;
START REPLICA;
SHOW REPLICA STATUS\G
```

Repeat if `Replica_SQL_Running` is still `No`. Each run skips one failed transaction.

> **Caution:** Skipping transactions can cause data inconsistencies between Master and Replica. Use this only in a controlled setup or during initial configuration. In production, investigate and resolve the root cause.

---

### Issue 3: `Plugin 'mysql_native_password' is not loaded`

**Symptom:** Error when trying to create a user with `mysql_native_password`.

**Cause:** MySQL 8.4 completely removed `mysql_native_password`. It is no longer available as an authentication plugin.

**Solution:** Use `caching_sha2_password` (the default) with `GET_SOURCE_PUBLIC_KEY = 1`:

```sql
CREATE USER 'replica'@'172.16.93.141' IDENTIFIED WITH caching_sha2_password BY 'Replica@123';
```

---

### Issue 4: MySQL fails to start after data directory change

**Symptom:** `systemctl start mysqld` fails after moving the data directory.

**Check the error log:**
```bash
sudo tail -50 /var/log/mysqld.log
```

**Most common cause — SELinux blocking access:**
```bash
# Verify SELinux context on the new directory
ls -lZ /data/mysql

# Re-apply the correct SELinux context
sudo semanage fcontext -a -t mysqld_db_t "/data/mysql(/.*)?"
sudo restorecon -Rv /data/mysql

sudo systemctl start mysqld
```

**Verify ownership:**
```bash
ls -la /data/ | grep mysql
# Should show: drwxr-x--- mysql mysql ... mysql
```

---

### Firewall verification

```bash
# On Master — verify firewall rule exists
sudo firewall-cmd --list-all

# From Replica — test connectivity to Master's MySQL port
nc -zv 172.16.93.140 3306
```

---

## 11. Quick Reference

### Key commands — Master

| Command | Purpose |
|---|---|
| `SHOW BINARY LOG STATUS\G` | Show current binlog file and position |
| `SHOW BINARY LOGS` | List all binlog files |
| `SHOW VARIABLES LIKE 'log_bin'` | Confirm binary logging is ON |
| `SHOW VARIABLES LIKE 'server_id'` | Confirm server ID |
| `SELECT user, host, plugin FROM mysql.user` | List all users with auth plugin |

### Key commands — Replica

| Command | Purpose |
|---|---|
| `SHOW REPLICA STATUS\G` | Full replication status and error details |
| `START REPLICA` | Start replication |
| `STOP REPLICA` | Stop replication |
| `RESET REPLICA ALL` | Clear all replication configuration |
| `SET GLOBAL SQL_REPLICA_SKIP_COUNTER = 1` | Skip one failed transaction |

### Key commands — Both servers

| Command | Purpose |
|---|---|
| `SHOW VARIABLES LIKE 'datadir'` | Confirm active data directory |
| `SHOW VARIABLES LIKE 'server_id'` | Confirm server ID |
| `SHOW PROCESSLIST` | Show active threads including replication threads |

---

### Replication health checklist

```
☐  Master: log_bin = ON
☐  Master: server-id = 1
☐  Master: replication user created with correct IP restriction
☐  Master: SHOW BINARY LOG STATUS — File and Position recorded
☐  Master: Firewall allows port 3306 from Replica IP
☐  Replica: server-id = 2 (different from Master)
☐  Replica: read_only = ON
☐  Replica: CHANGE REPLICATION SOURCE TO completed with GET_SOURCE_PUBLIC_KEY = 1
☐  Replica: START REPLICA executed
☐  Replica: Replica_IO_Running = Yes
☐  Replica: Replica_SQL_Running = Yes
☐  Replica: Seconds_Behind_Source = 0
☐  Live test: data written on Master appears on Replica
```

---

*MySQL 8.4 | Rocky Linux 9 | Tested and verified end-to-end*
