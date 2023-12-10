# Cinder

## Setup Database
```
mysql -e "CREATE DATABASE cinder;"
mysql -e "GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' IDENTIFIED BY 'password';"
mysql -e "GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' IDENTIFIED BY 'password';"
mysql -e "flush privileges;"
```

```
mysql -e "SELECT User, Db, Host from mysql.db;"
```

## Setup endpoint
```
openstack user create --domain default --password-prompt cinder
openstack role add --project service --user cinder admin
openstack service create --name cinderv3 --description "OpenStack Block Storage" volumev3

openstack endpoint create --region Region-JKT volumev3 public http://controller:8776/v3/%\(project_id\)s
openstack endpoint create --region Region-JKT volumev3 internal http://controller:8776/v3/%\(project_id\)s
openstack endpoint create --region Region-JKT volumev3 admin http://controller:8776/v3/%\(project_id\)s    
```

## Install and configure cinder
```
apt install cinder-api cinder-scheduler
```

```
vim /etc/cinder/cinder.conf
```

```
[database]
# ...
connection = mysql+pymysql://cinder:password@controller/cinder

[DEFAULT]
# ...
transport_url = rabbit://openstack:password@controller
auth_strategy = keystone
my_ip = 10.0.0.8

[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = password

[oslo_concurrency]
# ...
lock_path = /var/lib/cinder/tmp
```

```
vim /etc/nova/nova.conf
```

```
su -s /bin/sh -c "cinder-manage db sync" cinder
```

```
[cinder]
os_region_name = Region-JKT
```

```
systemctl restart nova-api
```

```
systemctl restart cinder-scheduler
systemctl restart apache2
```

```
openstack volume service list
```

## Create LVM Partition
```
apt install lvm2 thin-provisioning-tools
```

```
pvcreate /dev/sdb
vgcreate cinder-volumes /dev/sdb
```

```
pvs
vgs
```

## Install and Configure Cinder Volume
```
apt install cinder-volume tgt
```

```
vim /etc/cinder/cinder.conf
```

```
[DEFAULT]
# ...
enabled_backends = lvm
glance_api_servers = http://controller:9292

[lvm]
# ...
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volume_group = cinder-volumes
target_protocol = iscsi
target_helper = tgtadm
```

```
systemctl restart tgt
systemctl restart cinder-volume 
```

```
openstack volume service list
```