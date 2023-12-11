# Cinder

## Setup Database
Before you install and configure the Block Storage service, you must create a database, service credentials, and API endpoints
```
mysql -e "CREATE DATABASE cinder;"
mysql -e "GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' IDENTIFIED BY 'password';"
mysql -e "GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' IDENTIFIED BY 'password';"
mysql -e "flush privileges;"
```

Check user access database
```
mysql -e "SELECT User, Db, Host from mysql.db;"
```

## Setup endpoint
To create the service credentials, complete these steps
```
openstack user create --domain default --password-prompt cinder
openstack role add --project service --user cinder admin
openstack service create --name cinderv3 --description "OpenStack Block Storage" volumev3
```

Create the Block Storage service API endpoints
```
openstack endpoint create --region Region-JKT volumev3 public http://controller:8776/v3/%\(project_id\)s
openstack endpoint create --region Region-JKT volumev3 internal http://controller:8776/v3/%\(project_id\)s
openstack endpoint create --region Region-JKT volumev3 admin http://controller:8776/v3/%\(project_id\)s    
```

## Install and configure cinder
Install the packages
```
apt install cinder-api cinder-scheduler
```

Edit the /etc/cinder/cinder.conf file and complete the following actions
```
vim /etc/cinder/cinder.conf
```

Set this
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

Populate the Block Storage database:
```
su -s /bin/sh -c "cinder-manage db sync" cinder
```

Edit the /etc/nova/nova.conf file and add the following to it
```
vim /etc/nova/nova.conf
```

Set this
```
[cinder]
os_region_name = Region-JKT
```

Restart the Compute API service
```
systemctl restart nova-api
```

Restart the Block Storage services
```
systemctl restart cinder-scheduler
systemctl restart apache2
```

List service components to verify successful launch of each process
```
openstack volume service list
```

## Create LVM Partition
Install the supporting utility packages
```
apt install lvm2 thin-provisioning-tools
```

Create the LVM physical volume /dev/sdb
```
# create pvs
pvcreate /dev/sdb

# check pvs
pvs
```

Create the LVM volume group cinder-volumes
```
# create vgs
vgcreate cinder-volumes /dev/sdb

$ check vgs
vgs
```

Edit the /etc/lvm/lvm.conf file and complete the following actions
```
vim /etc/lvm/lvm.conf
```

Set this
```
devices {
...
filter = [ "a/sdb/", "r/.*/"] 
}
```

## Install and Configure Cinder Volume
Install the packages
```
apt install cinder-volume tgt
```

Edit the /etc/cinder/cinder.conf file and complete the following actions
```
vim /etc/cinder/cinder.conf
```

Set this
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

Restart the Block Storage volume service including its dependencies
```
systemctl restart tgt
systemctl restart cinder-volume 
```

List service components to verify successful launch of each process
```
openstack volume service list
```