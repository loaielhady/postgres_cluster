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
port = 9999        # Port number
backend_hostname0 = 'Primary server ip address'
                                   # Host name or IP address to connect to for backend 0
backend_port0 = 5432
                                   # Port number for backend 0
backend_weight0 = 1
                                   # Weight for backend 0 (only in load balancing mode)
backend_data_directory0 = '/var/lib/postgresql/16/main'
                                   # Data directory for backend 0
backend_flag0 = 'ALLOW_TO_FAILOVER'
                                            # Controls various backend behavior
                                          # ALLOW_TO_FAILOVER, DISALLOW_TO_FAILOVER
                                         # or ALWAYS_PRIMARY
#backend_application_name0 = 'server0'
                                       # walsender's application_name, used for "show pool_nodes" command
backend_hostname1 = 'standby server ip address '
backend_port1 = 5432
backend_weight1 = 1
backend_data_directory1 = '/var/lib/postgresql/16/main'
backend_flag1 = 'ALLOW_TO_FAILOVER'
backend_application_name1 = 'server1'

enable_pool_hba = on
auth_type = md5                                   # Use pool_hba.conf for client authentication
pool_passwd = '/etc/pgpool2/pool_passwd'
                                   # File name of pool_passwd for md5 authentication.
                                   # "" disables pool_passwd.
                                   # (change requires restart)
num_init_children = 32
max_pool = 4
                                   # Number of connection pool caches per connection
                                   # (change requires restart)
log_destination = 'stderr'
                                   # Where to log
                                   # Valid values are combinations of stderr,
                                   # and syslog. Default to stderr.
log_filename = 'pgpool.log'
                                        # log file name pattern,
                                        # can include strftime() escapes
load_balance_mode = on
                                   # Activate load balancing mode
sr_check_period = 10
                                   # Streaming replication check period
                                   # Default is 10s.
sr_check_user = 'postgres'
                                   # Streaming replication check user
                                   # This is necessary even if you disable streaming
                                   # replication delay check by sr_check_period = 0
sr_check_password = 'your_password'
                                   # Password for streaming replication check user
                                   # Leaving it empty will make Pgpool-II to first look for the
                                   # Password in pool_passwd file before using the empty password

sr_check_database = 'postgres'
health_check_period = 10
                                   # Health check period
                                   # Disabled (0) by default
health_check_timeout = 20
                                   # Health check timeout
                                   # 0 means no timeout
health_check_user = 'postgres'
                                   # Health check user
health_check_password = '1234'
                                   # Password for health check user
                                   # Leaving it empty will make Pgpool-II to first look for the
                                   # Password in pool_passwd file before using the empty password

health_check_database = 'postgres'
                                   # Database name for health check. If '', tries 'postgres' first.
health_check_max_retries = 3
                                   # Maximum number of times to retry a failed health check before giving up.
health_check_retry_delay = 5
failover_command = '/etc/pgpool2/failover.sh %d %P %H %M %R'
                                   # Executes this command at failover
                                   # Special values:
                                   #   %d = failed node id
                                   #   %h = failed node host name
                                   #   %p = failed node port number
                                   #   %D = failed node database cluster path
                                   #   %m = new main node id
                                   #   %H = new main node hostname
                                   #   %M = old main node id
                                   #   %P = old primary node id
                                   #   %r = new main port number
                                   #   %R = new main database cluster path
                                   #   %N = old primary node hostname
                                   #   %S = old primary node port number
                                   #   %% = '%' character
failback_command = '/etc/pgpool2/failover.sh %d %P %H %M %R'
                                   # Executes this command at failback.
                                   # Special values:
                                   #   %d = failed node id
                                   #   %h = failed node host name
                                   #   %p = failed node port number
                                   #   %D = failed node database cluster path
                                   #   %m = new main node id
                                   #   %H = new main node hostname
                                   #   %M = old main node id
                                   #   %P = old primary node id
                                   #   %r = new main port number
                                   #   %R = new main database cluster path
                                   #   %N = old primary node hostname
                                   #   %S = old primary node port number
                                   #   %% = '%' character

failover_on_backend_error = on
                                   # Initiates failover when reading/writing to the
                                   # backend communication socket fails
                                   # If set to off, pgpool will report an
                                   # error and disconnect the session.
hostname0 = ''
                                    # Host name or IP address of pgpool node

```


setting up a `pool_hba.conf` file

```
host    all         postgres    Primary_ip       md5
host    all         postgres    Standby_ip       md5
```

Script for log 

failback script:
```
#!/bin/bash 
FAILED_NODE_ID=$1
echo "Reataching node $FAILED_NODE_ID to pgpool-II after recovery."
exit 0
```
failover script:
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
