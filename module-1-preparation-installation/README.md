# Preparation Installation

## Set Hostname
Set hostname controller
```
hostnamectl set-hostname allinone-rafi
```

## Set Timezone
Set timezone to Asia/Jakarta
```
timedatectl set-timezone Asia/Jakarta

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
  ethernets:
    ens160:
      dhcp4: false
      addresses:
      - 10.0.0.8/24
      gateway4: 10.0.0.1
      nameservers:
        addresses:
        - 8.8.8.8
  version: 2
```

Load config
```
netplan apply
```

## Update & Upgrade
Update & upgrade repository
```
apt update -y && apt upgrade -y

apt autoremove -y && apt clean all
```

## Set /etc/host
Edit /etc/host
```
vim /etc/hosts
```

Set controller to host
```
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

allow 10.30.100.0/24
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
rabbitmqctl add_user openstack RABBIT_PASS
```

Add openstack user permission
```
rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

Verify
```
rabbitmqctl list_users

rabbitmqctl list_permissions
```

Check listening port and service
```
systemctl status rabbitmq-server

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
systemctl status memcached

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
systemctl status memcached

ss -tulpn
```