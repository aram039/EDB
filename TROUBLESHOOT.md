# EDB Cluster Management: Safe Shutdown/Startup & Barman Troubleshooting

A comprehensive guide for managing EDB Postgres clusters running on Vagrant VMs, with emphasis on safe cluster shutdown/startup procedures and barman backup troubleshooting.

---

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Part 1: Safe Shutdown & Startup](#part-1-safe-shutdown--startup)
  - [Shutdown Procedure](#shutdown-procedure)
  - [Startup Procedure](#startup-procedure)
  - [Verification Checklist](#verification-checklist)
- [Part 2: Barman Troubleshooting](#part-2-barman-troubleshooting)
  - [Overview](#barman-overview)
  - [The Issue We Solved](#the-issue-we-solved)
  - [Common Issues & Solutions](#common-issues--solutions)
  - [Best Practices](#best-practices)
  - [Daily Checklist](#daily-operational-checklist)
- [Quick Reference](#quick-reference)
- [Support & Contacts](#support--contacts)

---

## Overview

This guide is for managing an EDB Postgres cluster consisting of:
- **db-1**: Primary Postgres node
- **db-2**: Streaming replica
- **barman**: Backup server (pg_basebackup + WAL archiving)
- **efm-witness**: EFM witness node (optional)

Running on **Vagrant VMs** on a local laptop.

**Key Goals:**
- Safely shutdown the entire cluster before closing your laptop
- Safely startup the cluster after reopening your laptop
- Diagnose and fix barman backup failures
- Understand barman backup workflow and troubleshoot common issues

---

## Prerequisites

- Vagrant installed and VMs running (`vagrant status` shows all running)
- SSH config file at `~/.ssh/config` or use `ssh -F ssh_config` syntax
- Admin access to laptop/VMs
- Knowledge of basic PostgreSQL commands and `systemctl`

---

## Part 1: Safe Shutdown & Startup

### Why Safe Shutdown Matters

**Problem**: Stopping Vagrant VMs without gracefully stopping Postgres can cause:
- Unfinished transactions left in WAL
- Potential data corruption on startup
- Replication lag or divergence
- Failed backups in barman

**Solution**: Follow an ordered shutdown sequence that drains connections, finalizes backups, and stops services cleanly.

---

## Shutdown Procedure

### Step 1: Drain Connections (Primary db-1)

Terminate non-critical connections to allow a clean shutdown:

```bash
ssh -F ssh_config db-1 "psql -U postgres -c \"SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname NOT IN ('postgres', 'template0', 'template1') AND pid <> pg_backend_pid();\""
```

Verify replication is healthy:

```bash
ssh -F ssh_config db-1 "psql -U postgres -c \"SELECT * FROM pg_stat_replication;\""
```

**Expected output**: One row per replica showing `state = streaming`, zero lag.

**Output Format (Table View):**

| pid | usesysid | usename | application_name | client_addr | client_hostname | client_port | backend_start | state | sync_state | reply_time | flush_lsn | replay_lsn | write_lag | flush_lag | replay_lag |
|-----|----------|---------|------------------|-------------|-----------------|-------------|---------------|-------|-----------|------------|-----------|-----------|-----------|-----------|-----------|
| 1234 | 10 | streaming_barman | barman | 192.168.1.50 | barman | 55432 | 2026-06-05 18:20:15.123456+00 | streaming | async | 2026-06-05 18:25:00.123456+00 | 0/3000000 | 0/3000000 | 00:00:00.5 | 00:00:00.5 | 00:00:00.5 |

**Key columns to monitor:**
- **state**: Should be `streaming` (healthy) or `catchup` (replication lag)
- **sync_state**: `async` (asynchronous) or `sync` (synchronous)
- **client_addr**: IP of replica connecting
- **write_lag / flush_lag / replay_lag**: Should all be < 1 second for healthy replication

---

### Step 2: Take Final Backup (Barman)

On the barman host, trigger a final backup:

```bash
ssh -F ssh_config barman "barman backup db-1 --wait"
```

Verify success:

```bash
ssh -F ssh_config barman "barman list-backup db-1 | head -n 1"
```

**Expected output**: Latest backup shows `Done` status, recent timestamp, size > 0.

---

### Step 3: Stop Postgres (Replica First, then Primary)

Stop the replica node first to cleanly disconnect from primary:

```bash
ssh ec2-user@db-2 "sudo systemctl stop postgresql"
```

Wait 2 seconds, then stop the primary:

```bash
ssh ec2-user@db-1 "sudo systemctl stop postgresql"
```

Verify both stopped:

```bash
ssh ec2-user@db-1 "sudo systemctl status postgresql | head -n 3"
ssh ec2-user@db-2 "sudo systemctl status postgresql | head -n 3"
```

**Expected output**: `inactive (dead)` status.

---

### Step 4: Stop Barman Services (Optional)

Stop WAL archiver and barman to prevent timeouts during VM shutdown:

```bash
ssh -F ssh_config barman "sudo systemctl stop barman-receive-wal@db-1"
ssh -F ssh_config barman "sudo systemctl stop barman"
```

---

### Step 5: Shutdown Vagrant VMs

Now safe to shutdown:

```bash
# Graceful halt (recommended)
vagrant halt
```

Or force shutdown (if halt times out):

```bash
vagrant destroy -f
```

---

## Startup Procedure

### Step 1: Boot Vagrant VMs

```bash
vagrant up
```

Wait for VMs to fully boot (~1-2 minutes):

```bash
vagrant status
```

---

### Step 2: Verify Network & SSH

Test connectivity:

```bash
ssh -F ssh_config db-1 "echo OK"
ssh -F ssh_config db-2 "echo OK"
ssh -F ssh_config barman "echo OK"
```

**Expected output**: `OK` from each host.

---

### Step 3: Start Postgres (Primary First)

Start the primary:

```bash
ssh ec2-user@db-1 "sudo systemctl start postgresql"
sleep 2
ssh ec2-user@db-1 "sudo systemctl status postgresql | head -n 3"
```

---

### Step 4: Start Replica

Start the replica:

```bash
ssh ec2-user@db-2 "sudo systemctl start postgresql"
sleep 2
ssh ec2-user@db-2 "sudo systemctl status postgresql | head -n 3"
```

---

### Step 5: Start Barman Services

```bash
ssh -F ssh_config barman "sudo systemctl start barman-receive-wal@db-1"
ssh -F ssh_config barman "sudo systemctl start barman"
```

---

### Step 6: Verify Cluster Health

Check primary is writable:

```bash
psql -h db-1 -U postgres -c "SELECT pg_is_in_recovery();"
```

**Expected output**: `false`

Check replica is replaying:

```bash
psql -h db-2 -U postgres -c "SELECT pg_is_in_recovery(), pg_last_wal_replay_lsn();"
```

**Expected output**: `true | <LSN_value>`

Check replication:

```bash
psql -h db-1 -U postgres -c "SELECT * FROM pg_stat_replication;"
```

**Expected output**: 1+ row(s) with `state = streaming`.

Check barman:

```bash
ssh -F ssh_config barman "barman check db-1"
ssh -F ssh_config barman "barman list-backup db-1 | head -n 1"
```

**Expected output**: All checks OK, recent backup shown.

---

## Verification Checklist

Use this before shutdown and after startup:

- [ ] `pg_isready -h db-1 -p 5432` returns "accepting connections"
- [ ] `pg_isready -h db-2 -p 5432` returns "accepting connections"
- [ ] `psql -h db-1 -U postgres -c "SELECT * FROM pg_stat_replication;"` shows 1+ row
- [ ] `barman check db-1` shows no FAILED checks
- [ ] `barman list-backup db-1 | head -n1` shows recent backup
- [ ] `psql -h db-2 -U postgres -c "SELECT now() - pg_last_xact_replay_timestamp();"` shows < 1 second lag
- [ ] No errors in Postgres logs: `tail -n 50 /var/lib/pgsql/data/log/postgresql-*.log | grep ERROR`

---

## Part 2: Barman Troubleshooting

### Barman Overview

**barman** is a backup archiver for PostgreSQL. It:
1. Takes base backups using `pg_basebackup` (streaming)
2. Archives WAL (write-ahead logs) for point-in-time recovery (PITR)
3. Stores backups in `/var/lib/barman/db-1/base/<timestamp>`
4. Maintains a catalog of successful backups in `/var/lib/barman/db-1/meta`

**Backup workflow:**
```
Backup Start → pg_basebackup streams data → Backup label written → Backup complete → WALs continue archiving
```

---

### The Issue We Solved

#### Error Message

```
ERROR: The backup has failed copying files
ERROR: Backup failed writing backup label.
DETAILS: [Errno 2] No such file or directory: '/var/lib/barman/db-1/base/20260605T182103/data/backup_label'
```

#### Root Cause

The `rsync-concurrent` backup method had a **race condition**:
- `rsync` starts copying files from Postgres data directory
- Meanwhile, `pg_basebackup()` runs on the Postgres primary
- `backup_label` is created by `pg_stop_backup()`
- But `rsync` may finish before `backup_label` is created → file not copied → barman fails

**Why it happened**: The barman config had:
```
backup_method = rsync-concurrent  (default, not explicit)
reuse_backup = link               (global setting conflicting with rsync)
```

#### Solution Applied

1. **Changed backup method** from `rsync-concurrent` to `postgres` (uses `pg_basebackup` + `pg_receivexlog`):

```bash
sudo sed -i 's/^backup_method.*/backup_method = postgres/' /etc/barman.d/db-1.conf
```

2. **Disabled conflicting option**:

```bash
echo "reuse_backup = off" | sudo tee -a /etc/barman.d/db-1.conf
```

3. **Verified it worked**:

```bash
barman backup db-1
# Result: SUCCESS, 31.1 MiB backup created
```

4. **Cleaned up failed backups**:

```bash
barman delete db-1 20260605T171903
barman delete db-1 20260605T172403
```

---

### Common Issues & Solutions

#### Issue 1: Missing backup_label in rsync backup

**Symptoms**
```
ERROR: The backup has failed copying files
ERROR: Backup failed writing backup label.
DETAILS: [Errno 2] No such file or directory: '.../backup_label'
```

**How to diagnose**

Check barman logs:
```bash
sudo tail -n 100 /var/log/barman/barman.log | grep -E "ERROR|backup_label"
```

Check if rsync actually ran:
```bash
ls -la /var/lib/barman/db-1/base/
# If directories are present but empty → rsync failed/incomplete
```

Check Postgres logs for backup_label creation:
```bash
ssh ec2-user@db-1 "sudo tail -n 50 /var/lib/pgsql/data/log/postgresql-*.log | grep -E 'pg_stop_backup|backup_label|ERROR'"
```

**How to fix**

Switch to the `postgres` backup method (avoids rsync race):
```bash
sudo sed -i 's/backup_method.*/backup_method = postgres/' /etc/barman.d/db-1.conf
echo "reuse_backup = off" | sudo tee -a /etc/barman.d/db-1.conf
barman backup db-1 --wait
```

---

#### Issue 2: Replication slot missing or already exists

**Symptoms**
```
ERROR: Replication slot 'backup_barman' already exists
```

**How to diagnose**

Check current slots:
```bash
psql -h db-1 -U postgres -c "SELECT * FROM pg_replication_slots;"
```

**Expected output** (table format):

| slot_name | slot_type | datoid | database | temporary | active | active_pid | restart_lsn | confirmed_flush_lsn | wal_status | remain |
|-----------|-----------|--------|----------|-----------|--------|-----------|------------|---------------------|------------|--------|
| backup_barman | physical | | | f | t | 1234 | 0/3000000 | | reserved | |

Check barman config:
```bash
grep "slot_name" /etc/barman.d/db-1.conf
```

**How to fix**

Option A — Recreate the slot:
```bash
psql -h db-1 -U postgres -c "SELECT pg_drop_replication_slot('backup_barman');"
barman check db-1  # will recreate the slot
```

Option B — Use a different slot name:
```bash
sudo bash -c 'echo "slot_name = backup_barman_new" >> /etc/barman.d/db-1.conf'
barman check db-1
```

---

#### Issue 3: WAL archiving not progressing

**Symptoms**
```
barman check db-1 output:
  archiver errors: FAILED (WALs not being archived)
  wal size: HUGE (grows without bound)
  wal maximum age: FAILED
```

**How to diagnose**

Check Postgres archiver status:
```bash
psql -h db-1 -U postgres -c "SELECT * FROM pg_stat_archiver;"
```

**Expected output** (table format):

| archived_count | last_archived_wal | last_archived_time | failed_count | last_failed_wal | last_failed_time | stats_reset |
|---|---|---|---|---|---|---|
| 1234 | 000000010000000000000ABC | 2026-06-05 18:25:00.123456+00 | 0 | | | |

Check archive_command:
```bash
psql -h db-1 -U postgres -c "SHOW archive_command;"
```

Check barman can receive WALs:
```bash
ssh -F ssh_config barman "barman replication-status db-1"
```

Test SSH connectivity:
```bash
ssh -F ssh_config barman "ssh postgres@db-1 'psql -U streaming_barman -c \"SELECT 1;\"'"
```

**How to fix**

Ensure streaming_barman user has replication privileges:
```bash
psql -h db-1 -U postgres -c "ALTER ROLE streaming_barman WITH REPLICATION;"
```

Verify streaming_archiver is enabled in barman config:
```bash
grep "streaming_archiver" /etc/barman.d/db-1.conf
# should show: streaming_archiver = on
```

Restart Postgres if needed:
```bash
ssh ec2-user@db-1 "sudo systemctl restart postgresql"
```

Monitor archiving progress:
```bash
sleep 10
barman check db-1
```

---

#### Issue 4: Backup stuck in WAITING_FOR_WALS state

**Symptoms**
```
barman list-backup db-1 output:
  db-1 20260605T184225 - F - ...
  (status F = finalizing, waiting for WALs)
```

**How to diagnose**

Check if WALs are still arriving:
```bash
ssh -F ssh_config barman "ls -lart /var/lib/barman/db-1/incoming | tail -n 10"
```

Check if receive-wal process is running:
```bash
ssh -F ssh_config barman "ps aux | grep 'pg_receivewal|barman-receive'"
```

**How to fix**

Use the `--wait` flag to let barman wait for all WALs:
```bash
barman backup db-1 --wait
# waits indefinitely
```

Or with timeout:
```bash
barman backup db-1 --wait --wait-timeout 300  # 5 minutes
```

If stuck, force WAL segment switch:
```bash
barman switch-wal --force db-1
```

---

#### Issue 5: Barman cannot access Postgres PGDATA via rsync

**Symptoms**
```
ERROR: The backup has failed copying files
(no specific error, rsync fails silently or hangs)
```

**How to diagnose**

Test the SSH command barman uses:
```bash
ssh -F ssh_config barman "ssh -q postgres@db-1 -p 22 'ls -la /var/lib/pgsql/data' | head -n 20"
```

Test rsync directly:
```bash
ssh -F ssh_config barman "rsync -avz postgres@db-1:/var/lib/pgsql/data/PG_VERSION /tmp/test"
```

Check data directory permissions on db-1:
```bash
ssh ec2-user@db-1 "ls -ld /var/lib/pgsql/data"
# should be: drwx------ postgres postgres
```

Check barman has SSH keys:
```bash
ssh -F ssh_config barman "ls -la ~/.ssh/authorized_keys ~/.ssh/id_rsa"
```

**How to fix**

Ensure passwordless SSH from barman to db-1:

1. On barman, generate SSH key if missing:
```bash
ssh -F ssh_config barman "ssh-keygen -t rsa -N '' -f ~/.ssh/id_rsa || true"
```

2. Copy public key to db-1:
```bash
ssh -F ssh_config barman "cat ~/.ssh/id_rsa.pub" | ssh ec2-user@db-1 "sudo tee -a /home/postgres/.ssh/authorized_keys > /dev/null"
```

3. Fix permissions on db-1:
```bash
ssh ec2-user@db-1 "sudo chown postgres:postgres /home/postgres/.ssh/authorized_keys && sudo chmod 600 /home/postgres/.ssh/authorized_keys"
```

4. Verify:
```bash
ssh -F ssh_config barman "ssh -q postgres@db-1 'echo OK'"
# should output: OK
```

5. Retry backup:
```bash
barman backup db-1
```

---

#### Issue 6: Disk full on barman

**Symptoms**
```
ERROR: Backup failed ... ENOSPC (No space left on device)
```

**How to diagnose**

Check disk usage:
```bash
ssh -F ssh_config barman "df -h /var/lib/barman"
ssh -F ssh_config barman "du -sh /var/lib/barman/db-1"
```

List failed backups:
```bash
ssh -F ssh_config barman "barman list-backup db-1 | grep 'FAILED\|INCOMPLETE'"
```

Check backup sizes:
```bash
ssh -F ssh_config barman "barman list-backup db-1 | awk '{print \$1, \$2, \$NF}'"
```

**How to fix**

Delete failed/orphan backups:
```bash
ssh -F ssh_config barman "barman delete db-1 <backup-id>"
```

Delete oldest backups if storage exceeded:
```bash
ssh -F ssh_config barman "barman list-backup db-1 | tail -n 5 | awk '{print \$2}' | while read id; do barman delete db-1 \$id; done"
```

If persistent, expand storage:
```bash
ssh -F ssh_config barman "df -h"
# consider adding another disk or extending current volume
```

---

### Best Practices

#### Backup Method: rsync vs postgres

| Aspect | rsync-concurrent | postgres |
|--------|------------------|----------|
| Method | Copies files directly via rsync | Uses pg_basebackup + pg_receivexlog |
| Speed | Faster (parallel streams) | Slower but more reliable |
| Race conditions | Possible (backup_label) | None (streaming-based) |
| Reuse capability | Yes (hardlinks) | No (needs reuse_backup = off) |
| **Recommended** | **No** | **Yes** |

**Recommendation**: Use `backup_method = postgres` for reliability. Only use `rsync` if you have verified it works in your environment and need faster backups.

---

#### Configuration Best Practices

**barman.conf** (global):
```ini
[barman]
backup_method = rsync-concurrent
reuse_backup = link

[db-1]  # or in /etc/barman.d/db-1.conf
backup_method = postgres
reuse_backup = off
```

**Why**: Override global settings per server to avoid conflicts.

---

#### Retention Policy

Set automatic retention to prevent disk fill:

```bash
echo "retention_policy = 'RECOVERY WINDOW OF 7 DAYS'" | sudo tee -a /etc/barman.d/db-1.conf
```

Run cron to apply retention:
```bash
barman cron
```

---

### Daily Operational Checklist

**Morning (after startup):**
```bash
# 1. Cluster health
psql -h db-1 -U postgres -c "SELECT * FROM pg_stat_replication;"

# 2. Barman status
barman check db-1

# 3. Last backup age (should be < 24h)
barman list-backup db-1 | head -n1

# 4. Disk space
df -h /var/lib/barman
```

**Evening (before shutdown):**
```bash
# 1. Take final backup
barman backup db-1 --wait

# 2. Verify it succeeded
barman list-backup db-1 | head -n1

# 3. Check no archiver errors
psql -h db-1 -U postgres -c "SELECT * FROM pg_stat_archiver WHERE failed_count > 0;"

# 4. Shutdown sequence
# (follow "Shutdown Procedure" section above)
```

---

## Quick Reference

### Essential Commands

**Connectivity**
```bash
pg_isready -h db-1 -p 5432
psql -h db-1 -U postgres -c "SELECT 1;"
```

**Replication**
```bash
psql -h db-1 -U postgres -c "SELECT * FROM pg_stat_replication;"
psql -h db-2 -U postgres -c "SELECT pg_is_in_recovery();"
```

**Barman Backups**
```bash
barman backup db-1 --wait
barman list-backup db-1
barman show-backup db-1 <backup-id>
barman delete db-1 <backup-id>
```

**Service Control**
```bash
ssh ec2-user@db-1 "sudo systemctl {start|stop|status|restart} postgresql"
ssh -F ssh_config barman "sudo systemctl {start|stop|status|restart} barman"
```

---

## Known Issues Summary

| Issue | Cause | Solution |
|-------|-------|----------|
| `backup_label` missing | rsync race condition | Use `backup_method = postgres` |
| `reuse_backup` conflict | Incompatible with postgres method | Add `reuse_backup = off` to server config |
| Replication slot exists | Incomplete delete | Drop slot and recreate: `pg_drop_replication_slot(...)` |
| WAL not archiving | Missing streaming_barman role or perms | Grant REPLICATION: `ALTER ROLE streaming_barman WITH REPLICATION;` |
| Backup WAITING_FOR_WALS | Normal; waiting for WAL archival | Use `--wait` flag or `barman switch-wal --force` |
| rsync fails | SSH/rsync not configured | Set up passwordless SSH, verify `ssh postgres@db-1` works |
| Disk full | Backups not purged or storage exhausted | Delete old backups, expand storage |

---

## Support & Contacts

**On-call DBA**: [fill in]  
**DBA Lead**: [fill in]  
**Infra Lead**: [fill in]  
**EDB Support**: [contract & portal details]

**Escalation path**: On-call → DBA Lead → Infra Lead → EDB Support

---

## Additional Resources

- **EDB Docs**: https://www.enterprisedb.com/docs
- **PostgreSQL Docs**: https://www.postgresql.org/docs
- **Barman Docs**: https://www.pgbarman.org
- **Vagrant Docs**: https://www.vagrantup.com/docs

---

## Document Info

- **Last Updated**: June 6, 2026
- **Cluster**: db-1 (primary), db-2 (replica), barman, efm-witness
- **Platform**: Vagrant VMs
- **Status**: Production-ready

---

**Questions?** See the [Common Issues & Solutions](#common-issues--solutions) section or contact your DBA team.
