# Nova & Placement

## Setup Database placement
```
mysql -e "CREATE DATABASE placement;"
mysql -e "GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' IDENTIFIED BY 'password';"
mysql -e "GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' IDENTIFIED BY 'password';"
mysql -e "flush privileges;"
```

```
mysql -e "SELECT User, Db, Host from mysql.db;"
```

## Setup endpoint placement
```
openstack user create --domain default --password-prompt placement
openstack role add --project service --user placement admin
openstack service create --name placement --description "Placement API" placement

openstack endpoint create --region Region-JKT placement public http://controller:9292
openstack endpoint create --region Region-JKT placement internal http://controller:9292
openstack endpoint create --region Region-JKT placement admin http://controller:9292
```

## Install and Configuration Placement
```
apt install placement-api
```

```
vim /etc/placement/placement.conf
```

```
[placement_database]
# ...
connection = mysql+pymysql://placement:password@controller/placement

[api]
# ...
auth_strategy = keystone

[keystone_authtoken]
# ...
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = placement
password = password
```

```
su -s /bin/sh -c "placement-manage db sync" placement
```

```
service apache2 restart
```

```
placement-status upgrade check
```

## Setup Database Nova
```
mysql -e "CREATE DATABASE nova_api;"
mysql -e "CREATE DATABASE nova;"
mysql -e "CREATE DATABASE nova_cell0;"
mysql -e "GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY 'password';"
mysql -e "GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'password';"
mysql -e "GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'password';"
mysql -e "GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'password';"
mysql -e "GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY 'password';"
mysql -e "GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'password';"
mysql -e "flush privileges;"
```

```
mysql -e "SELECT User, Db, Host from mysql.db;"
```

## Setup endpoint Nova
```
openstack user create --domain default --password-prompt nova
openstack role add --project service --user nova admin
openstack service create --name nova --description "OpenStack Compute" compute

openstack endpoint create --region Region-JKT compute public http://controller:9292
openstack endpoint create --region Region-JKT compute internal http://controller:9292
openstack endpoint create --region Region-JKT compute admin http://controller:9292
```

## Install and Configuration Nova
```
apt install nova-api nova-conductor nova-novncproxy nova-scheduler
```

```
vim /etc/nova/nova.conf
```

```
[api_database]
# ...
connection = mysql+pymysql://nova:password@controller/nova_api

[database]
# ...
connection = mysql+pymysql://nova:password@controller/nova

[DEFAULT]
# ...
transport_url = rabbit://openstack:password@controller:5672/

[api]
# ...
auth_strategy = keystone
my_ip = 10.0.0.8

[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000/
auth_url = http://controller:5000/
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = password

[service_user]
send_service_user_token = true
auth_url = https://controller/identity
auth_strategy = keystone
auth_type = password
project_domain_name = Default
project_name = service
user_domain_name = Default
username = nova
password = password

[vnc]
enabled = true
# ...
server_listen = $my_ip
server_proxyclient_address = $my_ip

[glance]
# ...
api_servers = http://controller:9292

[oslo_concurrency]
# ...
lock_path = /var/lib/nova/tmp

[placement]
# ...
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = password
```

```
su -s /bin/sh -c "nova-manage api_db sync" nova
```

```
su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
```

```
su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
```

```
su -s /bin/sh -c "nova-manage db sync" nova
```

```
su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
```

```
systemctl restart nova-{api,scheduler,conductor,novncproxy}
```