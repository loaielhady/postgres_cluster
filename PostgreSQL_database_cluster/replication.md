# Replication in PostgreSQL

This document describes a simple primary → standby replication setup for PostgreSQL. The primary handles writes and one or more standbys replicate the WAL stream and can serve read-only queries for read scaling or failover.

## Prerequisites
- PostgreSQL installed on primary and standby.
- Network connectivity port 5432 from standby to primary.

## Overview of the steps
1. Configure primary (`postgresql.conf` and `pg_hba.conf`) to allow replication.
2. Create a replication role on the primary.
3. Take a base backup from the primary into standby.
4. Start the standby and verify replication.

---

## Primary server configuration

```conf
# Listen on all addresses (or list specific IPs)
listen_addresses = '*'

# WAL and replication settings
wal_level = replica                 # or 'logical' if using logical replication
max_wal_senders = 10
max_replication_slots = 10          # if you plan to use replication slots
wal_keep_size = '64MB'              # prevents removal of recent WAL files
password_encryption = md5

```
## postgres user database configuration

```
$ sudo -u postgres psql
```

Show password encryption :


```
SHOW password_encryption;

```

Set encryption password:


```
ALTER SYSTEM SET password_encryption = 'md5';

```

Add replication user:

```
CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD  'your_password';

```
Create user with password:

```
ALTER USER postgres WITH PASSWORD  'your_password';

```

### pg_hba.conf changes (primary)

```
host    replication     replicator      your.ip.address/..         md5

```

### last step in primary server is

```
$ sudo systemctl restart postgresql

```

## Standby setup (base backup method)

1. Stop PostgreSQL on the standby:

```bash
sudo systemctl stop postgresql
```

2. Remove or move existing data directory on the standby :

```bash
sudo -i -u postgres bash
rm -rf /var/lib/postgresql/16/main/*
exit
```

3. Take a base backup from the primary using `pg_basebackup`. Run this on the standby as the `postgres` user:

```bash
# run as postgres user
sudo -u postgres pg_basebackup -h PRIMARY_IP \
  -D /var/lib/postgresql/16/main \
  -U replicator -v -P -R -X stream -S node_name
```

Notes:
- `-R` writes a `standby.signal` file and `primary_conninfo` into `postgresql.auto.conf` , enabling automatic connection to primary on startup.
- `-X stream` streams WAL with the base backup.
- `-S node_name` creates a replication slot with the given name.
- Ensure the destination directory is owned by the `postgres` user and has correct permissions:

```bash
sudo chown -R postgres:postgres /var/lib/postgresql/16/main
```

4. Start PostgreSQL on the standby:

```bash
sudo systemctl start postgresql
```

Because of `-R`, the standby will have `standby.signal` and will attempt to connect to the primary automatically.

---

## Verifying replication

On the primary:

```
SELECT * from pg_replication_slots;

```

On the standby:

```sql
-- returns true if this server is in recovery
SELECT pg_is_in_recovery();

-- show current replication connection info
SELECT * FROM pg_stat_wal_receiver;
```
