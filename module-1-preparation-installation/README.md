# Preparation Installation

## Set Hostname
Set hostname controller
```
hostnamectl set-hostname allinone-rafi
```

## Set Timezone
Set timezone to Asia/Jakarta
```
# set timezone to WIB
timedatectl set-timezone Asia/Jakarta

# check timezone
date
```

## Set Static IP
Edit netplan config
```
vim /etc/netplan/00-installer-config.yaml
```

Set static ip address
```
network:
  version: 2
  renderer: networkd
  ethernets:
    ens160:
      addresses:
        - 10.0.0.8/24
      routes:
        - to: default
          via: 10.0.0.1
      nameservers:
          search: [google.com]
          addresses: [1.1.1.1, 8.8.8.8]
```

Load config
```
netplan apply
```

## Update ,Upgrade, and Cleaning Packages
Update repository & Update packages
```
apt update -y && apt upgrade -y
```

Cleaning package
```
apt autoremove -y && apt clean all
```

## Set /etc/host
Edit /etc/host
```
vim /etc/hosts
```

Set controller to host
```
# openstack environtment
10.0.0.8 controller
```

## NTP Server
Install chrony
```
apt install chrony
```

Config chrony
```
vim /etc/chrony/chrony.conf
```

Set this 
```
server 0.id.pool.ntp.org
server 1.id.pool.ntp.org
server 2.id.pool.ntp.org
server 3.id.pool.ntp.org

allow 10.0.0.0/24
```

Restart service chrony
```
systemctl restart chrony
```

Verify chrony
```
chronyc sources
```

## Openstack Package
OpenStack Zed for Ubuntu 22.04 LTS:
```
add-apt-repository cloud-archive:zed
```

Openstack client installation
```
apt install python3-openstackclient
```

## SQL database
Install the packages:
```
apt install mariadb-server python3-pymysql
```

Config mariadb
```
vim /etc/mysql/mariadb.conf.d/99-openstack.cnf
```

Set this
```
[mysqld]
bind-address = 10.0.0.8

default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8    
```

Restart the database service:
```
systemctl restart mariadb
```    

Secure database service
```
mysql_secure_installation
```

## Message Queue Server
Install the packages
```
apt install rabbitmq-server
```

Add openstack user
```
rabbitmqctl add_user openstack password
```

Add openstack user permission
```
rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

Verify
```
# list user
rabbitmqctl list_users

# list user with permission
rabbitmqctl list_permissions

# test authentication
rabbitmqctl authenticate_user openstack password
```

Check listening port and service
```
# check service
systemctl status rabbitmq-server

# check port listening
ss -tulpn
```

## Memcached
Install the packages
```
apt install memcached python3-memcache
```

Config memcached
```
vim /etc/memcached.conf 
```

Set this
```
-l 10.0.0.8
```

Restart the Memcached service
```
systemctl restart memcached
```

Check listening port and service
```
# check service
systemctl status memcached

# check port listening
ss -tulpn
```

## Etcd
Install the packages
```
apt install etcd
```

Config etcd
```
vim /etc/default/etcd
```

Set this
```
ETCD_NAME="controller"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"
ETCD_INITIAL_CLUSTER="controller=http://10.0.0.8:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.0.0.8:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://10.0.0.8:2379"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_LISTEN_CLIENT_URLS="http://10.0.0.8:2379"
```

Restart etcd
```
systemctl restart etcd
```

Check listening port and service
```
# check service
systemctl status etcd

# check port listening
ss -tulpn
```