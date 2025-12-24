# Pgpool

Pgpool is an open-source middleware that sits between PostgreSQL databases and client applications, acting as a proxy to provide features like load balancing, connection pooling, and automatic failover for high availability and scalability. It transparently handles database requests, directing reads to multiple servers and writes to a master, significantly boosting performance by reducing connection overhead and distributing workload without requiring major application code changes. 

## Prerequisites
- pgpool-II installed on server
- connectivety between primary and standby server to pgpool-II server.

## Overview of the steps
1. configuration servers `pg_hba.conf` to allow connectivity to pgpool-II
2. Configure primary (`pgpool.conf` , `cpc.conf` and `pool.hba.conf`) to allow pgpool-II work.


setup ip of pgpool to all servers primary and standby.

in that path 
```
sudo nano /etc/postgresql/16/main/pg_hba.conf
```


```
host        all      all     pgpool.ip.address/       md5
```

```
listen_addresses = '*'
port = 9999        
backend_hostname0 = 'Primary server ip address'
backend_port0 = 5432
backend_weight0 = 1              
backend_data_directory0 = '/var/lib/postgresql/16/main'   
backend_flag0 = 'ALLOW_TO_FAILOVER'
backend_hostname1 = 'standby server ip address '
backend_port1 = 5432
backend_weight1 = 1
backend_data_directory1 = '/var/lib/postgresql/16/main'
backend_flag1 = 'ALLOW_TO_FAILOVER'
backend_application_name1 = 'server1'
enable_pool_hba = on
auth_type = md5                                 
pool_passwd = '/etc/pgpool2/pool_passwd'           
num_init_children = 32
max_pool = 4
log_destination = 'stderr'
log_filename = 'pgpool.log'
load_balance_mode = on
sr_check_period = 10
sr_check_user = 'postgres'
sr_check_password = 'your_password'
sr_check_database = 'postgres'
health_check_period = 10 
health_check_timeout = 20
health_check_user = 'postgres'
health_check_password = '1234'
health_check_database = 'postgres'
health_check_max_retries = 3
health_check_retry_delay = 5
failover_command = '/etc/pgpool2/failover.sh %d %P %H %M %R'
failback_command = '/etc/pgpool2/failover.sh %d %P %H %M %R'
failover_on_backend_error = on
hostname0 = ''
```


setting up a `pool_hba.conf` file

```
host    all         postgres    Primary_ip       md5
host    all         postgres    Standby_ip       md5
```

Script for log 

### failback script:
```
#!/bin/bash 
FAILED_NODE_ID=$1
echo "Reataching node $FAILED_NODE_ID to pgpool-II after recovery."
exit 0
```

### failover script:

```
#!/bin/bash
FAILED_NODE_ID=$1
NEW_PRIMARY_HOST=$3

if [ $FAILED_NODE_ID -eq 0 ]; then
	echo "Failover detected. Promoting standby (node 1) to primary..."
	psql -h $NEW_PRIMARY_HOST -U postgres -c "SELECT pg_promote();"
fi
exit 0
```
change owner of file by:
```
sudo chown /etc/pgpool2/failover.sh
sudo chown /etc/pgpool2/failback.sh
```

```
sudo mkdir -p /var/log/pgpool
sudo chown postgres:postgres /var/log/pgpool
sudo systemctl restart pgpool2
```

### start pgpool

```
sudo start pgpool2
sudo enable pgpool2
```

### Test pgpool 

from any client enter

```
psql -h <pgpool-ip>   -p  9999  -U postgres
```
```sql
SHOW pool_nodes;
```
