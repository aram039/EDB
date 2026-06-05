```markdown
# EDB Postgres — Cluster Maintenance, Disk Resizing & Troubleshooting Guide

## For EC2-Based Deployments (EDB Postgres Advanced Server + EFM + Barman)

> **Covers:** EDB Postgres Advanced Server (EPAS) · EDB Failover Manager (EFM) v5.x · Barman  
> **Environment:** AWS EC2 instances (RHEL / Amazon Linux / Ubuntu)  
> **Sources:**  
> - https://www.enterprisedb.com/docs  
> - https://www.enterprisedb.com/docs/efm/latest/  
> - https://docs.aws.amazon.com/whitepapers/latest/optimizing-postgresql-on-ec2-using-ebs/  
> **Last Updated:** June 2026

---

## Table of contents

1. [Architecture Overview](#1-architecture-overview)  
2. [Key Paths & Service Names](#2-key-paths--service-names)  
3. [Cluster Maintenance](#3-cluster-maintenance)  
   - [Pre-Maintenance Checklist](#pre-maintenance-checklist)  
   - [Planned Switchover (EFM)](#planned-switchover-efm)  
   - [Rolling OS Patching](#rolling-os-patching)  
   - [Postgres Configuration Changes](#postgres-configuration-changes)  
4. [Disk & EBS Resizing (AWS EC2)](#4-disk--ebs-resizing-aws-ec2)  
   - [Resize EBS Volume (No Downtime)](#resize-ebs-volume-no-downtime)  
   - [Extend the Filesystem](#extend-the-filesystem)  
   - [Add a Separate WAL Volume](#add-a-separate-wal-volume)  
   - [Monitoring Disk Usage](#monitoring-disk-usage)  
5. [Troubleshooting — EDB Postgres Database](#5-troubleshooting--edb-postgres-database)  
6. [Troubleshooting — EFM (Failover Manager)](#6-troubleshooting--efm-failover-manager)  
7. [Troubleshooting — Barman](#7-troubleshooting--barman)  
8. [AWS-Specific Troubleshooting](#8-aws-specific-troubleshooting)  
9. [Quick Reference — Commands Cheat Sheet](#9-quick-reference--commands-cheat-sheet)  
10. [References](#10-references)

---

## 1. Architecture Overview

A typical EDB HA cluster on AWS EC2 looks like this:

```

┌─────────────────────────────────────────────────────────────────────┐

│                         AWS Region (e.g. ap-south-1)                │

│                                                                     │

│   ┌──────────────────┐    Streaming     ┌──────────────────┐        │

│   │  EC2 - Primary   │──────────────── ▶│  EC2 - Standby   │        │

│   │  EPAS + EFM Agent│  Replication     │  EPAS + EFM Agent│        │

│   │  AZ: ap-south-1a │                  │  AZ: ap-south-1b │        │

│   └──────────────────┘                  └──────────────────┘        │

│            │                                      │                 │

│            │           ┌──────────────────┐       │                 │

│            └──────────▶│  EC2 - Witness   │◀──────┘                 │

│                        │  EFM Agent only  │                         │

│                        │  AZ: ap-south-1c │                         │

│                        └──────────────────┘                         │

│                                                                     │

│   ┌──────────────────┐                                              │

│   │  EC2 - Barman    │◀─── WAL Archiving / Base Backups ────────────│

│   │  (Backup Server) │                                              │

│   └──────────────────┘                                              │

└─────────────────────────────────────────────────────────────────────┘

```

**Component Roles:**

| Component | Role |
|---|---|
| **EPAS (Primary)** | Accepts read/write connections; streams WAL to standby |
| **EPAS (Standby)** | Receives WAL, ready for promotion; can serve read-only |
| **EFM Agent** | Monitors database health on each node; orchestrates failover |
| **Witness Node** | Tiebreaker for quorum — no database, EFM agent only |
| **Barman Server** | Dedicated backup server; stores base backups and WAL archive |
| **VIP (Virtual IP)** | Floats to the current primary; applications connect to this |

---

## 2. Key Paths & Service Names

> Adjust version numbers (e.g. `15`, `16`) to match your installation.

```

# ── EDB Postgres Advanced Server ───────────────────────────────────

PGDATA=/var/lib/edb/as16/data

PGLOG=/var/log/edb/as16/

PG_CONF=/var/lib/edb/as16/data/postgresql.conf

PG_HBA=/var/lib/edb/as16/data/pg_hba.conf

PG_SERVICE=edb-as-16

# ── EFM ────────────────────────────────────────────────────────────

EFM_HOME=/usr/edb/efm-5.x

EFM_PROPS=/etc/edb/efm-5.x/[efm.properties](http://efm.properties)

EFM_CLUSTER_PROPS=/etc/edb/efm-5.x/efm.cluster

EFM_LOG=/var/log/efm-5.x/

EFM_SERVICE=edb-efm-5.x

# ── Barman ─────────────────────────────────────────────────────────

BARMAN_HOME=/var/lib/barman

BARMAN_CONF=/etc/barman.conf

BARMAN_SERVER_CONF=/etc/barman.d/<server>.conf

BARMAN_LOG=/var/log/barman/barman.log

```

**Service control (systemd):**

```

# Postgres

sudo systemctl start|stop|restart|status edb-as-16

# EFM

sudo systemctl start|stop|restart|status edb-efm-5.x

# Barman (runs as 'barman' user, not a daemon — jobs via cron)

sudo crontab -u barman -l

```

---

## 3. Cluster Maintenance

### Pre-Maintenance Checklist

```

# 1. Confirm cluster health

sudo /usr/edb/efm-5.x/bin/efm cluster-status efm

# 2. Check replication is in sync

sudo -u enterprisedb psql -c "SELECT client_addr, state, sent_lsn,

write_lsn, flush_lsn, replay_lsn,

pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes

FROM pg_stat_replication;"

# 3. Verify Barman backups are current

barman check all

barman list-backup all

# 4. Confirm disk space on all nodes

df -h /var/lib/edb/

# 5. Check for any active long-running transactions

sudo -u enterprisedb psql -c "

SELECT pid, now() - pg_stat_activity.query_start AS duration,

query, state

FROM pg_stat_activity

WHERE (now() - pg_stat_activity.query_start) > interval '5 minutes'

ORDER BY duration DESC;"

```

---

### Planned Switchover (EFM)

```

sudo /usr/edb/efm-5.x/bin/efm cluster-status efm

sudo /usr/edb/efm-5.x/bin/efm set-priority efm <standby-ip> 1

sudo /usr/edb/efm-5.x/bin/efm promote efm -switchover

sudo /usr/edb/efm-5.x/bin/efm cluster-status efm

```

---

### Rolling OS Patching

```

sudo systemctl stop edb-efm-5.x

sudo dnf update -y     # RHEL/Amazon Linux

# OR

sudo apt-get upgrade -y  # Ubuntu/Debian

sudo reboot

sudo systemctl start edb-as-16

sudo systemctl start edb-efm-5.x

sudo /usr/edb/efm-5.x/bin/efm cluster-status efm

```

---

### Postgres Configuration Changes

```

sudo -u enterprisedb vim /var/lib/edb/as16/data/postgresql.conf

sudo -u enterprisedb psql -c "

SELECT name, setting, unit, context

FROM pg_settings

WHERE context = 'postmaster';"

sudo systemctl reload edb-as-16

# OR

sudo -u enterprisedb psql -c "SELECT pg_reload_conf();"

sudo systemctl restart edb-as-16

```

---

## 4. Disk & EBS Resizing (AWS EC2)

### Resize EBS Volume (No Downtime)

```

lsblk

df -h

aws ec2 describe-volumes \

--filters "Name=attachment.instance-id,Values=<your-instance-id>" \

--query "Volumes[*].{ID:VolumeId,Size:Size,Device:Attachments[0].Device}"

aws ec2 modify-volume \

--volume-id vol-0xxxxxxxxxxxxxxxxx \

--size 200

aws ec2 describe-volumes-modifications \

--volume-id vol-0xxxxxxxxxxxxxxxxx \

--query "VolumesModifications[0].ModificationState"

lsblk

```

---

### Extend the Filesystem

#### For ext4 filesystem:

```

df -T /var/lib/edb

sudo growpart /dev/xvdb 1

sudo resize2fs /dev/xvdb1

# OR if no partition:

sudo resize2fs /dev/xvdb

df -h /var/lib/edb

```

#### For XFS filesystem:

```

sudo xfs_growfs /var/lib/edb

df -h /var/lib/edb

```

---

### Add a Separate WAL Volume

```

sudo mkfs.ext4 /dev/xvdc

sudo mkdir -p /mnt/pgwal

echo "/dev/xvdc  /mnt/pgwal  ext4  defaults,nofail  0 2" | sudo tee -a /etc/fstab

sudo mount -a

sudo systemctl stop edb-as-16

sudo mv /var/lib/edb/as16/data/pg_wal /mnt/pgwal/

sudo chown -R enterprisedb:enterprisedb /mnt/pgwal/

sudo -u enterprisedb ln -s /mnt/pgwal/pg_wal /var/lib/edb/as16/data/pg_wal

sudo systemctl start edb-as-16

ls -la /var/lib/edb/as16/data/pg_wal

```

---

### Monitoring Disk Usage

```

df -h

sudo du -sh /var/lib/edb/as16/data/* | sort -rh | head -20

sudo du -sh /var/lib/edb/as16/data/pg_wal/

sudo -u enterprisedb psql -c "

SELECT datname,

pg_size_pretty(pg_database_size(datname)) AS size

FROM pg_database

ORDER BY pg_database_size(datname) DESC;"

sudo -u enterprisedb psql -d mydb -c "

SELECT schemaname, tablename,

pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size

FROM pg_tables

ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC

LIMIT 20;"

```

---

## 5. Troubleshooting — EDB Postgres Database

### Scenario 1 — Database Won't Start

```

sudo systemctl status edb-as-16

sudo tail -100 /var/log/edb/as16/*.log

sudo journalctl -u edb-as-16 -n 100 --no-pager

ls -la /var/lib/edb/as16/data/[postmaster.pid](http://postmaster.pid)

sudo -u enterprisedb cat /var/lib/edb/as16/data/[postmaster.pid](http://postmaster.pid)

sudo ss -tlnp | grep 5444

ls -la /var/lib/edb/as16/data/

df -h /var/lib/edb/

```

**Common log messages and fixes:**

| Log Message | Cause | Fix |
|---|---|---|
| `could not bind IPv4 address: Address already in use` | Port conflict | Kill the conflicting process |
| `data directory has wrong ownership` | Permission issue | `chown -R enterprisedb:enterprisedb $PGDATA` |
| `could not open file "global/pg_control"` | Corrupt data directory | Restore from Barman backup |
| `lock file "postmaster.pid" already exists` | Stale PID | Remove PID file if process is dead |
| `out of memory` | Insufficient RAM | Reduce `shared_buffers` |

---

### Scenario 2 — Disk Full / PGDATA Out of Space

```

df -h

sudo du -sh /var/lib/edb/as16/data/* | sort -rh | head -10

sudo -u enterprisedb psql -c "SELECT pg_switch_wal();"

du -sh /var/lib/edb/as16/data/pg_wal/

sudo find /var/log/edb/ -name "*.log" -mtime +7 -delete

sudo systemctl status edb-as-16

sudo systemctl start edb-as-16

sudo -u enterprisedb psql -d mydb -c "

SELECT schemaname, tablename,

pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size

FROM pg_tables

ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC

LIMIT 10;"

```

---

### Scenario 3 — Replication Lag / Standby Behind Primary

```

sudo -u enterprisedb psql -c "

SELECT client_addr,

application_name,

state,

pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn)   AS sent_lag_bytes,

pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS replay_lag_bytes,

sync_state

FROM pg_stat_replication;"

sudo -u enterprisedb psql -c "

SELECT now() - pg_last_xact_replay_timestamp() AS replication_delay;"

sudo -u enterprisedb psql -c "SELECT pg_is_in_recovery(), pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn();"

```

---

### Scenario 4 — Connections Refused / Max Connections Hit

```

sudo -u enterprisedb psql -c "

SELECT count(*) AS total,

count(*) FILTER (WHERE state = 'active')  AS active,

count(*) FILTER (WHERE state = 'idle')    AS idle,

count(*) FILTER (WHERE state = 'idle in transaction') AS idle_in_tx

FROM pg_stat_activity;"

sudo -u enterprisedb psql -c "SHOW max_connections;"

sudo -u enterprisedb psql -c "

SELECT usename, application_name, client_addr, state, count(*)

FROM pg_stat_activity

GROUP BY usename, application_name, client_addr, state

ORDER BY count(*) DESC;"

sudo -u enterprisedb psql -c "

SELECT pg_terminate_backend(pid)

FROM pg_stat_activity

WHERE state = 'idle'

AND query_start < now() - interval '10 minutes'

AND pid <> pg_backend_pid();"

sudo -u enterprisedb psql -c "ALTER SYSTEM SET max_connections = 300;"

sudo systemctl restart edb-as-16

```

---

### Scenario 5 — Long-Running Queries Blocking

```

sudo -u enterprisedb psql -c "

SELECT pid,

now() - query_start AS runtime,

usename,

wait_event_type,

wait_event,

left(query, 120) AS query

FROM pg_stat_activity

WHERE state = 'active'

AND now() - query_start > interval '1 minute'

ORDER BY runtime DESC;"

sudo -u enterprisedb psql -c "

SELECT [bl.pid](http://bl.pid)         AS blocked_pid,

a.usename      AS blocked_user,

ka.query       AS blocking_query,

[ka.pid](http://ka.pid)         AS blocking_pid,

a.query        AS blocked_query

FROM pg_[catalog.pg](http://catalog.pg)_locks bl

JOIN pg_[catalog.pg](http://catalog.pg)_stat_activity a  ON [a.pid](http://a.pid)  = [bl.pid](http://bl.pid)

JOIN pg_[catalog.pg](http://catalog.pg)_locks kl         ON kl.transactionid = bl.transactionid

AND [kl.pid](http://kl.pid) != [bl.pid](http://bl.pid)

JOIN pg_[catalog.pg](http://catalog.pg)_stat_activity ka ON [ka.pid](http://ka.pid) = [kl.pid](http://kl.pid)

WHERE NOT bl.granted;"

sudo -u enterprisedb psql -c "SELECT pg_cancel_backend(<pid>);"

sudo -u enterprisedb psql -c "SELECT pg_terminate_backend(<pid>);"

```

---

### Scenario 6 — Table / Index Bloat

```

sudo -u enterprisedb psql -d mydb -c "

SELECT schemaname, tablename,

pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size,

n_dead_tup,

n_live_tup,

round(n_dead_tup::numeric / NULLIF(n_live_tup + n_dead_tup, 0) * 100, 2) AS dead_pct,

last_autovacuum,

last_vacuum

FROM pg_stat_user_tables

ORDER BY n_dead_tup DESC

LIMIT 20;"

sudo -u enterprisedb psql -d mydb -c "VACUUM VERBOSE ANALYZE mytable;"

sudo -u enterprisedb psql -d mydb -c "VACUUM FULL VERBOSE ANALYZE mytable;"

sudo -u enterprisedb psql -d mydb -c "REINDEX INDEX CONCURRENTLY myindex;"

```

---

### Scenario 7 — WAL Accumulation / Disk Fill from WAL

```

du -sh /var/lib/edb/as16/data/pg_wal/

sudo -u enterprisedb psql -c "

SELECT slot_name, active,

pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) AS lag_bytes

FROM pg_replication_slots

ORDER BY lag_bytes DESC;"

sudo -u enterprisedb psql -c "SELECT pg_drop_replication_slot('stale_slot');"

ls /var/lib/edb/as16/data/pg_wal/archive_status/*.ready | wc -l

sudo -u enterprisedb psql -c "SHOW wal_keep_size;"

sudo -u enterprisedb psql -c "ALTER SYSTEM SET wal_keep_size = '1GB';"

sudo -u enterprisedb psql -c "SELECT pg_reload_conf();"

```

---

### Scenario 8 — Autovacuum Issues

```

sudo -u enterprisedb psql -c "SHOW autovacuum;"

sudo -u enterprisedb psql -c "

SELECT datname,

age(datfrozenxid) AS xid_age,

pg_size_pretty(pg_database_size(datname)) AS db_size

FROM pg_database

ORDER BY age(datfrozenxid) DESC;"

sudo -u enterprisedb psql -d mydb -c "

SELECT schemaname, relname,

age(relfrozenxid) AS xid_age,

pg_size_pretty(pg_relation_size(oid)) AS size

FROM pg_class

WHERE relkind = 'r'

ORDER BY age(relfrozenxid) DESC

LIMIT 20;"

sudo -u enterprisedb psql -d mydb -c "VACUUM FREEZE VERBOSE mytable;"

sudo -u enterprisedb psql -c "

SELECT pid, usename, wait_event, query

FROM pg_stat_activity

WHERE query LIKE 'autovacuum%';"

```

---

### Scenario 9 — Database Corruption / Crash Recovery

```

sudo grep -i "invalid|corrupt|wrong checksum" /var/log/edb/as16/*.log

sudo systemctl status edb-as-16

sudo -u enterprisedb psql -c "SELECT pg_check_all_tables();"

sudo -u enterprisedb pg_dump -Fc mydb > /tmp/mydb_dump_check.dump 2>&1

```

---

## 6. Troubleshooting — EFM (Failover Manager)

### EFM Cluster Status Overview

```

sudo /usr/edb/efm-5.x/bin/efm cluster-status efm

sudo tail -100 /var/log/efm-5.x/efm-<clustername>.log

sudo journalctl -u edb-efm-5.x -n 100 --no-pager

```

---

### Scenario 1 — EFM Agent Fails to Start

```

sudo journalctl -u edb-efm-5.x -n 50 --no-pager

sudo cat /var/log/efm-5.x/startup-efm.log

java -version

sudo dnf install java-17-openjdk -y

sudo vi /etc/sysconfig/efm-5.x

# JAVA_EXECUTABLE_PATH=/usr/lib/jvm/java-17-openjdk/bin/java

sudo dnf install tzdata-java -y

sudo /usr/edb/efm-5.x/bin/efm validate efm

sudo ss -tlnp | grep 7800

sudo grep "bind.address" /etc/edb/efm-5.x/[efm.properties](http://efm.properties)

```

---

### Scenario 2 — Primary Database Crashes (Auto-Failover)

```

sudo /usr/edb/efm-5.x/bin/efm cluster-status efm

sudo -u enterprisedb psql -h <new-primary-ip> -c "SELECT pg_is_in_recovery();"

sudo grep "promoted|promotion|failover" /var/log/efm-5.x/efm-efm.log

```

---

### Scenario 3 — Primary EC2 Instance Goes Down

```

ping <vip-address>

sudo /usr/edb/efm-5.x/bin/efm cluster-status efm

sudo ip addr show | grep <vip-address>

sudo -u enterprisedb psql -c "SELECT count(*) FROM pg_stat_activity WHERE state='active';"

```

---

### Scenario 4 — Split-Brain / Network Isolation

```

sudo grep "isolated|released VIP|network partition" /var/log/efm-5.x/efm-efm.log

sudo systemctl stop edb-as-16

```

---

### Scenario 5 — Standby Agent or Node Fails

```

sudo /usr/edb/efm-5.x/bin/efm cluster-status efm

sudo systemctl status edb-as-16

sudo tail -50 /var/log/edb/as16/*.log

sudo systemctl status edb-efm-5.x

sudo systemctl start edb-as-16

sudo systemctl start edb-efm-5.x

sudo /usr/edb/efm-5.x/bin/efm resume efm

```

---

### Scenario 6 — Witness Node Failure

```

sudo /usr/edb/efm-5.x/bin/efm cluster-status efm

sudo systemctl start edb-efm-5.x

sudo /usr/edb/efm-5.x/bin/efm cluster-status efm

```

---

### Scenario 7 — Manual Switchover (Planned)

```

sudo /usr/edb/efm-5.x/bin/efm cluster-status efm

sudo /usr/edb/efm-5.x/bin/efm set-priority efm <standby-ip> 1

sudo -u enterprisedb psql -c "

SELECT pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes

FROM pg_stat_replication

WHERE client_addr = '<standby-ip>';"

sudo /usr/edb/efm-5.x/bin/efm promote efm -switchover

sudo /usr/edb/efm-5.x/bin/efm cluster-status efm

sudo -u enterprisedb psql -h <new-primary-ip> -c "SELECT pg_is_in_recovery();"

```

---

### Scenario 8 — Re-joining Old Primary as Standby

```

sudo systemctl stop edb-as-16

sudo -u enterprisedb pg_rewind \

--target-pgdata=/var/lib/edb/as16/data \

--source-server="host=<new-primary-ip> port=5444 user=enterprisedb dbname=postgres" \

--progress

sudo -u enterprisedb touch /var/lib/edb/as16/data/standby.signal

cat >> /var/lib/edb/as16/data/[postgresql.auto](http://postgresql.auto).conf << 'EOF'

primary_conninfo = 'host=<new-primary-ip> port=5444 user=replicator password=<password>'

EOF

sudo systemctl start edb-as-16

sudo /usr/edb/efm-5.x/bin/efm resume efm

# Option B: pg_basebackup

sudo systemctl stop edb-as-16

sudo -u enterprisedb rm -rf /var/lib/edb/as16/data/*

sudo -u enterprisedb pg_basebackup \

-h <new-primary-ip> \

-p 5444 \

-U replicator \

-D /var/lib/edb/as16/data \

--wal-method=stream \

-P -R

sudo systemctl start edb-as-16

sudo /usr/edb/efm-5.x/bin/efm resume efm

```

---

### Scenario 9 — EFM VIP Not Moving After Failover

```

ip addr show | grep <vip>

sudo grep -i "vip|virtual|script" /var/log/efm-5.x/efm-efm.log | tail -50

sudo /usr/edb/efm-5.x/bin/efm assign-virtual-ip efm <vip-address>

ls -la /usr/edb/efm-5.x/bin/efm_address

sudo /usr/edb/efm-5.x/bin/efm_address add <interface> <vip/prefix>

# e.g.

sudo /usr/edb/efm-5.x/bin/efm_address add eth0 10.0.1.100/24

```

---

### Common EFM Errors & Fixes

| Error Message | Cause | Fix |
|---|---|---|
| `Authorization file not found. Is the local agent running?` | EFM agent is not running | `systemctl start edb-efm-5.x` |
| `Not authorized to run this command. User 'X' is not a member of the efm group` | Wrong user | `usermod -aG efm username` |
| `OutOfMemory` in EFM log | JVM heap too small | Increase heap in efm.properties |
| `JGRP000006: failed accepting connection` | External traffic to EFM port | Restrict SG/firewall for 7800 |
| `discarded message from different cluster` | Old config cached | `efm reset-members efm` |
| `java.io.FileNotFoundException: tzdb.dat` | Missing tzdata-java | `dnf install tzdata-java` |

---

## 7. Troubleshooting — Barman

### Barman Check & Status

```

barman check all

barman check postgres_primary

barman list-backup postgres_primary

barman show-backup postgres_primary latest

barman switch-wal postgres_primary

barman check postgres_primary

```

---

### Scenario 1 — WAL Archiving Failing

```

ls -la /var/lib/edb/as16/data/pg_wal/archive_status/*.ready | wc -l

sudo -u enterprisedb ssh barman@<barman-server-ip> "barman receive-wal --test postgres_primary"

sudo -u enterprisedb psql -c "SHOW archive_command;"

sudo -u enterprisedb barman-wal-archive <barman-host> postgres_primary \

/var/lib/edb/as16/data/pg_wal/archive_status/../0000000100000001000000AB

sudo tail -100 /var/log/barman/barman.log | grep -i "error|fail|warn"

barman check postgres_primary

sudo -u barman psql -h <postgres-primary-ip> -U barman -d postgres -c "SELECT 1;"

sudo -u enterprisedb psql -c "SELECT pg_switch_wal();"

```

---

### Scenario 2 — Backup Failing

```

sudo -u barman barman backup postgres_primary --wait

sudo -u barman ssh enterprisedb@<postgres-ip>

sudo -u barman ssh-keygen -t rsa -b 4096

sudo -u barman ssh-copy-id enterprisedb@<postgres-ip>

df -h /var/lib/barman/

sudo -u barman psql -h <postgres-ip> -U barman -d postgres

sudo grep "pg_basebackup|ERROR" /var/log/barman/barman.log | tail -30

barman check postgres_primary

barman backup postgres_primary

```

---

### Scenario 3 — Barman SSH Connectivity Issues

```

sudo -u barman ssh -v enterprisedb@<postgres-ip>

sudo -u enterprisedb ssh -v barman@<barman-ip>

sudo -u barman ssh -o StrictHostKeyChecking=no enterprisedb@<postgres-ip>

sudo grep "ssh_command" /etc/barman.d/postgres_primary.conf

barman check postgres_primary | grep ssh

```

---

### Scenario 4 — Point-in-Time Recovery (PITR)

```

barman list-backup postgres_primary

sudo systemctl stop edb-as-16

sudo systemctl stop edb-efm-5.x

sudo mv /var/lib/edb/as16/data /var/lib/edb/as16/data_corrupt_$(date +%Y%m%d%H%M)

sudo mkdir -p /var/lib/edb/as16/data

sudo chown enterprisedb:enterprisedb /var/lib/edb/as16/data

sudo -u barman barman recover \

--remote-ssh-command "ssh enterprisedb@<postgres-ip>" \

--target-time "2026-06-05 14:34:00 UTC" \

--get-wal \

postgres_primary \

20260605T020000 \

/var/lib/edb/as16/data

sudo -u enterprisedb cat /var/lib/edb/as16/data/[postgresql.auto](http://postgresql.auto).conf

sudo systemctl start edb-as-16

sudo tail -f /var/log/edb/as16/*.log

sudo -u enterprisedb psql -c "SELECT max(created_at) FROM orders;"

sudo -u enterprisedb psql -c "SELECT pg_promote();"

sudo systemctl start edb-efm-5.x

```

---

### Scenario 5 — Barman Disk Full

```

df -h /var/lib/barman/

barman list-backup all

barman delete postgres_primary oldest

barman delete postgres_primary 20260501T020000

barman cron

df -h /var/lib/barman/

barman check postgres_primary

```

---

### Scenario 6 — WAL Gap / Missing WAL Files

```

barman check postgres_primary

barman show-server postgres_primary | grep -i wal

barman list-files postgres_primary latest | grep -i wal

sudo -u enterprisedb psql -c "

SELECT pg_current_wal_lsn(), pg_walfile_name(pg_current_wal_lsn());"

sudo -u enterprisedb psql -c "SELECT pg_switch_wal();"

sudo -u barman barman backup postgres_primary

barman switch-wal --force --archive postgres_primary

barman check postgres_primary

```

---

## 8. AWS-Specific Troubleshooting

### EC2 Instance Type Constraints

```

aws cloudwatch get-metric-statistics \

--namespace AWS/EBS \

--metric-name VolumeQueueLength \

--dimensions Name=VolumeId,Value=vol-xxxxxxxxx \

--start-time 2026-06-05T00:00:00Z \

--end-time 2026-06-05T12:00:00Z \

--period 300 \

--statistics Average

aws ec2 modify-volume \

--volume-id vol-xxxxxxxxx \

--volume-type gp3 \

--iops 6000 \

--throughput 500

```

### Security Groups — Required Ports

| Source | Destination | Port | Protocol | Purpose |
|---|---|---:|---|---|
| App servers | Primary/Standby | 5444 | TCP | EPAS connections |
| Primary | Standby | 5444 | TCP | Streaming replication |
| All DB/Witness nodes | All DB/Witness nodes | 7800-7810 | TCP | EFM agent communication |
| Barman server | DB nodes | 22 | TCP | SSH for WAL archiving |
| DB nodes | Barman server | 22 | TCP | SSH (barman recover) |
| DB nodes | Barman server | 5444 | TCP | Barman Postgres monitoring |
| Admin | All nodes | 22 | TCP | SSH admin access |

```

nc -zv <target-ip> 7800

nc -zv <barman-ip> 22

nc -zv <primary-ip> 5444

```

### AWS-Specific Failover with EFM + Elastic IP

```

# post.promotion.script=/usr/local/bin/reassign_[eip.sh](http://eip.sh)

#!/bin/bash

INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)

ALLOCATION_ID="eipalloc-xxxxxxxxxxxxxxxxx"

aws ec2 associate-address \

--instance-id $INSTANCE_ID \

--allocation-id $ALLOCATION_ID \

--allow-reassociation

logger "EFM: Elastic IP $ALLOCATION_ID reassigned to $INSTANCE_ID after promotion"

```

---

## 9. Quick Reference — Commands Cheat Sheet

```

# EFM

efm cluster-status efm

efm promote efm -switchover

efm set-priority efm <standby-ip> 1

efm resume efm

efm stop-cluster efm

efm reset-members efm

efm assign-virtual-ip efm <vip>

efm validate efm

# EPAS

systemctl start|stop|restart|status edb-as-16

sudo -u enterprisedb psql

# Replication

SELECT * FROM pg_stat_replication;

SELECT pg_is_in_recovery();

SELECT now() - pg_last_xact_replay_timestamp();

SELECT pg_wal_replay_resume();

# Connections

SELECT count(*), state FROM pg_stat_activity GROUP BY state;

SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE ...;

# Disk / WAL

SELECT pg_switch_wal();

SELECT pg_drop_replication_slot('name');

SELECT * FROM pg_replication_slots;

# Maintenance

VACUUM VERBOSE ANALYZE <table>;

VACUUM FREEZE VERBOSE <table>;

REINDEX INDEX CONCURRENTLY <index>;

SELECT pg_reload_conf();

# Barman

barman check all

barman check <server>

barman list-backup <server>

barman show-backup <server> latest

barman backup <server>

barman delete <server> oldest

barman switch-wal <server>

barman cron

barman recover --target-time "..." <server> <backup-id> <dest>

# Disk

df -h

lsblk

du -sh /var/lib/edb/as16/data/* | sort -rh

sudo growpart /dev/xvdb 1

sudo resize2fs /dev/xvdb1

sudo xfs_growfs /mountpoint

# AWS EBS

aws ec2 describe-volumes --filters "Name=attachment.instance-id,Values=<id>"

aws ec2 modify-volume --volume-id <vol-id> --size 200

aws ec2 describe-volumes-modifications --volume-id <vol-id>

aws ec2 modify-volume --volume-id <vol-id> --volume-type gp3 --iops 6000

```

---

## 10. References

- EDB EFM Documentation (v5.x): https://www.enterprisedb.com/docs/efm/latest/  
- EFM Supported Failover Scenarios: https://www.enterprisedb.com/docs/efm/latest/11_supported_scenarios/  
- EFM Troubleshooting: https://www.enterprisedb.com/docs/efm/latest/13_troubleshooting/  
- EFM efm Utility Reference: https://www.enterprisedb.com/docs/efm/latest/07_using_efm_utility/  
- Barman Documentation: https://www.enterprisedb.com/docs/supported-open-source/barman/  
- Barman PITR Guide: https://www.enterprisedb.com/docs/supported-open-source/barman/single-server-streaming/  
- EDB Postgres Advanced Server Docs: https://www.enterprisedb.com/docs/epas/latest/  
- AWS — Optimizing PostgreSQL on EC2 + EBS: https://docs.aws.amazon.com/whitepapers/latest/optimizing-postgresql-on-ec2-using-ebs/  
- EDB Blog: HA with EFM Tutorial: https://www.enterprisedb.com/blog/high-availability-enterprise-failover-manager-tutorial  
- EDB Blog: EFM + JDBC Failover Scenarios: https://www.enterprisedb.com/blog/improve-database-resiliency-new-jdbc-connectors-and-edb-failover-manager  
- EDB Knowledge Base: https://knowledge.enterprisedb.com/

> **Disclaimer:** This document is based on EDB official documentation, AWS documentation, and field best practices. Always verify steps in a staging environment before applying to production.
```
