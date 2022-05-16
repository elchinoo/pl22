# Patroni

_"Patroni is a template for you to create your own customized, high-availability solution using Python and - for maximum accessibility - a distributed configuration store like ZooKeeper, etcd, Consul or Kubernetes. Database engineers, DBAs, DevOps engineers, and SREs who are looking to quickly deploy HA PostgreSQL in the datacenter-or anywhere else-will hopefully find it useful"_. (https://github.com/zalando/patroni)

We'll use ETCD as our DCS in this tutorial.

## Installing ETCD

### Node-1

The first step is to install ETCD on *node_1*:

```bash
yum install -y etcd

# Backup the configuration file 
mv /etc/etcd/etcd.conf /etc/etcd/etcd.conf.bak

# Create the new configuration file
cat <<EOF > /etc/etcd/etcd.conf
#[Member]
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="http://192.168.21.60:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.21.60:2379,http://localhost:2379"
ETCD_NAME="node-1"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.21.60:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.21.60:2379"
ETCD_INITIAL_CLUSTER="node-1=http://192.168.21.60:2380"
ETCD_INITIAL_CLUSTER_TOKEN="PL2022-etcd-cluster-01"
ETCD_INITIAL_CLUSTER_STATE="new"
EOF

# Enable and start ETCD service
systemctl enable etcd
systemctl start etcd
systemctl status etcd
```

### Node-2 and node-3

Now that we have the first node running ETCD we need to proceed in the other nodes. The steps will be the same for both nodes but they need to be run sequentially:

```bash
# First thing is to add the node-2 to the cluster. We need to run the below command on NODE-1
etcdctl member add node-2 http://192.168.21.61:2380

# We use the returned information to create the configuration file for node-2
# Backup the configuration file 
mv /etc/etcd/etcd.conf /etc/etcd/etcd.conf.bak

# Create the new configuration file
cat <<EOF > /etc/etcd/etcd.conf
#[Member]
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="http://192.168.21.61:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.21.61:2379,http://localhost:2379"
ETCD_NAME="node-2"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.21.61:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.21.61:2379"
ETCD_INITIAL_CLUSTER="node-1=http://192.168.21.60:2380,node-2=http://192.168.21.61:2380"
ETCD_INITIAL_CLUSTER_TOKEN="PL2022-etcd-cluster-01"
ETCD_INITIAL_CLUSTER_STATE="existing"
EOF

# Enable and start ETCD service
systemctl enable etcd
systemctl start etcd
systemctl status etcd

####
### We then repeat the same process to node-3
##
# Add the node-3 to the cluster. We need to run the below command on NODE-1
etcdctl member add node-3 http://192.168.21.62:2380

# We use the returned information to create the configuration file for node-3
# Backup the configuration file 
mv /etc/etcd/etcd.conf /etc/etcd/etcd.conf.bak

# Create the new configuration file
cat <<EOF > /etc/etcd/etcd.conf
#[Member]
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="http://192.168.21.62:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.21.62:2379,http://localhost:2379"
ETCD_NAME="node-3"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.21.62:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.21.62:2379"
ETCD_INITIAL_CLUSTER="node-3=http://192.168.21.62:2380,node-1=http://192.168.21.60:2380,node-2=http://192.168.21.61:2380"
ETCD_INITIAL_CLUSTER_TOKEN="PL2022-etcd-cluster-01"
ETCD_INITIAL_CLUSTER_STATE="existing"
EOF

# Enable and start ETCD service
systemctl enable etcd
systemctl start etcd
systemctl status etcd
```

## Patroni

### Node-1
We start with *node-1*:

```bash
yum install -y python3 python3-pip
yum install -y patroni patroni-etcd

# Let's check where is the patroni configuration located based on our systemd service file
grep "ExecStart" /usr/lib/systemd/system/patroni.service

# In my case I got:
# ExecStart=/usr/bin/patroni /etc/patroni/patroni.yml
# The start command passes the file "/etc/patroni/patroni.yml" to patroni
# We probably need to create the folder "/etc/patroni"

mkdir /etc/patroni

cat <<EOF > /etc/patroni/patroni.yml
scope: PL22-PGHA
namespace: PL22
name: node-1

restapi:
    listen: 192.168.21.60:8008
    connect_address: 192.168.21.60:8008

etcd:
    host: 192.168.21.60:2379

bootstrap:
    dcs:
        ttl: 30
        loop_wait: 10
        retry_timeout: 10
        maximum_lag_on_failover: 1048576
        # synchronous_mode: true
        postgresql:
            use_pg_rewind: true
            use_slots: true
            parameters:
                wal_level: replica
                hot_standby: "on"
                wal_keep_segments: 8
                max_wal_senders: 5
                max_replication_slots: 10
                wal_log_hints: "on"
                archive_mode: "on"
                archive_timeout: 1800s
                archive_command: "/bin/true"
                # recovery_conf:
                #    restore_command: cp ../wal_archive/%f %p
        initdb:
        - encoding: UTF8
        - data-checksums

        pg_hba:
        - host replication replicator 127.0.0.1/32 md5
        - host replication replicator 192.168.21.60/0 md5
        - host replication replicator 192.168.21.61/0 md5
        - host replication replicator 192.168.21.62/0 md5
        - host all all 0.0.0.0/0 md5

    users:
        admin:
            password: admin
            options:
                - createrole
                - createdb

postgresql:
    listen: 192.168.21.60:5432
    connect_address: 192.168.21.60:5432
    data_dir: /var/lib/pgsql/14/data/
    bin_dir: /usr/pgsql-14/bin
    pgpass: /tmp/pgpass
    authentication:
        replication:
            username: replicator
            password: replicator
        superuser:
            username: postgres
            password: postgres
#    parameters:
#        unix_socket_directories: "/var/run/postgresql/"
#
#    callbacks:
#        on_reload: /opt/app/patroni/etc/callback.sh
#        on_restart: /opt/app/patroni/etc/callback.sh
#        on_role_change: /opt/app/patroni/etc/callback.sh
#        on_start: /opt/app/patroni/etc/callback.sh
#        on_stop: /opt/app/patroni/etc/callback.sh

#watchdog:
#    mode: automatic # Allowed values: off, automatic, required
#    device: /dev/watchdog
#    safety_margin: 5

tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false

EOF

systemctl enable patroni 
systemctl restart patroni
systemctl status patroni
patronictl -c /etc/patroni/config.yml list armorblox-cluster1
```

### Nodes 2 and 3
Now we can add the replicas one by one using the above configuration template but making sure we change the relevant parameter values.