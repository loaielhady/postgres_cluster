# Replication in postgresSQL

Primary-secondary replication is a common setup in PostgreSQL cluster, where the primary database handles all write operations, while the standby handles read operations.
This allows for read scalability, as multiple replicas can distribute the load of read queries across the cluster, reducing the performance impact on the primary database.

## Setup for replication in Primary server :-

```
listen_addresses = '*'				
wal_level = replica
max_wal_senders = 10
wal_keep_size = 64	
hot_standby = on
password_encryption = md5

```

## postgres 

```
$ sudo -u postgres psql
```

add password:

```
SHOW password_encryption;

```

### set encryption password:


```
ALTER SYSTEM SET password_encryption = 'md5';

```

add replication user:

```
CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD  'your_password';

```
### create user with password:

```
ALTER USER postgres WITH PASSWORD  'your_password';

```

## pg_hba.conf editing in master:

```
host    replication     replicator      your.ip.address/..         md5

```

### last step in primary server is

```
$ sudo systemctl restart postgresql

```

### setting in Standby

```
$ sudo systemctl stop postgresql

```
### swich to user postgres 

```
$ sudo -i -u postgres bash
```

remove main file from standby server
```
$ sudo rm -rf /var/lib/postgresql/16/main/

```
#### copy files from primary server:

```
pg_basebackup -h your.primary.server.ip/address  -D /var/lib/postgres/16/main/  -U replicator -P -v -R -X stream -C -S node_name

```

### Restart postgres and test :

```
SELECT * from pg_replication_slots;

```


