# Floating IP address with Keepalived

Floating IP address is used to support failover in a high-availability cluster. The cluster is configured such that only the active member of the cluster "owns" or responds to that IP address at any given time. Should the active member fail, then "ownership" of the floating IP address would be transferred to a standby member to promote it as the new active member. Specifically, the member to be promoted issues a gratuitous ARP, announcing the new MAC address–to–IP address association.

_"The main goal of keepalived project is to provide simple and robust facilities for loadbalancing and high-availability to Linux system and Linux based infrastructures. Loadbalancing framework relies on well-known and widely used Linux Virtual Server (IPVS) kernel module providing Layer4 loadbalancing"_. More information here https://github.com/acassen/keepalived

Here we will configure a Floating IP address for our 3 nodes cluster of Postgres servers using Keepalived. Find the steps below:

```bash
yum install -y keepalived
echo "net.ipv4.ip_nonlocal_bind = 1" >> /etc/sysctl.conf
sysctl -p
cd /etc/keepalived/
mv keepalived.conf keepalived.conf.bak

# Keepalived configuration file on Primary
cat <<EOF > keepalived.conf
! Configuration File for keepalived

global_defs {
    notification_email {
        root@node-1
    }
    notification_email_from root@node-1
    smtp_server 127.0.0.1
    smtp_connect_timeout 30
    router_id LVS_DEVEL
}

vrrp_instance VI_1 {
    state MASTER
    interface eth1
    virtual_router_id 51
    priority 101 #used in election, 101 for primary & 100 for replicas
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.21.100/24
    }
}
EOF


## Keepalived replica node2
cat <<EOF > keepalived.conf
! Configuration File for keepalived

global_defs {
    notification_email {
        root@node-2
    }
    notification_email_from root@node-2
    smtp_server 127.0.0.1
    smtp_connect_timeout 30
    router_id LVS_DEVEL
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth1
    virtual_router_id 51
    priority 100 #used in election, 101 for primary & 100 for replicas
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.21.100/24
    }
}
EOF

## Keepalived replica node3
cat <<EOF > keepalived.conf
! Configuration File for keepalived

global_defs {
    notification_email {
        root@node-3
    }
    notification_email_from root@node-3
    smtp_server 127.0.0.1
    smtp_connect_timeout 30
    router_id LVS_DEVEL
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth1
    virtual_router_id 51
    priority 99 #used in election, 101 for primary & 100 for replicas
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.21.100/24
    }
}
EOF


systemctl start keepalived
systemctl enable keepalived

```

At this point we can halt node-1 with `vagrant halt node-1` and test our configuration!