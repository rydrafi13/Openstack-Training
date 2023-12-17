# Nova & Placement

## Setup Database placement
Before you install and configure the placement service, you must create a database, service credentials, and API endpoints.
```
mysql -e "CREATE DATABASE placement;"
mysql -e "GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' IDENTIFIED BY 'password';"
mysql -e "GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' IDENTIFIED BY 'password';"
mysql -e "flush privileges;"
```

Check user access database
```
mysql -e "SELECT User, Db, Host from mysql.db;"
```

## Setup endpoint placement
To create the service credentials, complete these steps
```
openstack user create --domain default --password-prompt placement
openstack role add --project service --user placement admin
openstack service create --name placement --description "Placement API" placement
```

Create the Placement service API endpoints:
```
openstack endpoint create --region JKT-01 placement public http://controller:8778
openstack endpoint create --region JKT-01 placement internal http://controller:8778
openstack endpoint create --region JKT-01 placement admin http://controller:8778
```

## Install and Configuration Placement
Install the packages
```
apt install placement-api
```

Edit the /etc/placement/placement.conf file and complete the following actions
```
vim /etc/placement/placement.conf
```

Set this
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

Populate the placement database
```
su -s /bin/sh -c "placement-manage db sync" placement
```

Reload the web server to adjust to get new configuration settings for placement
```
service apache2 restart
```

Perform status checks to make sure everything is in order
```
placement-status upgrade check
```

Run some commands against the placement API
<br>
Install the osc-placement plugin
```
apt install python3-pip
pip3 install osc-placement
```

List available resource classes and traits
```
openstack --os-placement-api-version 1.2 resource class list --sort-column name
openstack --os-placement-api-version 1.6 trait list --sort-column name
```

## Setup Database Nova
Before you install and configure the Compute service, you must create databases, service credentials, and API endpoints
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

Check user access database
```
mysql -e "SELECT User, Db, Host from mysql.db;"
```

## Setup endpoint Nova
Create the Compute service credentials
```
openstack user create --domain default --password-prompt nova
openstack role add --project service --user nova admin
openstack service create --name nova --description "OpenStack Compute" compute
```

Create the Compute API service endpoints
```
openstack endpoint create --region JKT-01 compute public http://controller:8774/v2.1
openstack endpoint create --region JKT-01 compute internal http://controller:8774/v2.1
openstack endpoint create --region JKT-01 compute admin http://controller:8774/v2.1
```

## Install and Configuration Nova
Install the packages
```
apt install nova-api nova-conductor nova-novncproxy nova-scheduler
```

Edit the /etc/nova/nova.conf file and complete the following actions
```
vim /etc/nova/nova.conf 
```

Set this
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
instances_path=/var/lib/nova/instances
my_ip = 10.0.0.18

[api]
# ...
auth_strategy = keystone

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
auth_url = http://controller:5000/identity
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
server_listen = 0.0.0.0
server_proxyclient_address = $my_ip
novncproxy_base_url = http://10.0.0.18:6080/vnc_auto.html

[glance]
# ...
api_servers = http://controller:9292

[oslo_concurrency]
# ...
lock_path = /var/lib/nova/tmp

[placement]
# ...
region_name = JKT-01
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = password
```

Populate the nova-api database
```
su -s /bin/sh -c "nova-manage api_db sync" nova
```

Register the cell0 database
```
su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
```

Create the cell1 cell
```
su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
```

Populate the nova database
```
su -s /bin/sh -c "nova-manage db sync" nova
```

Verify nova cell0 and cell1 are registered correctly
```
su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
```

Restart the Compute services
```
systemctl restart nova-{api,scheduler,conductor,novncproxy}
```

List service components to verify successful launch and registration of each process
```
openstack compute service list
```

## Install and Configuration Nova Compute
Install the packages
```
apt install nova-compute
```

Determine whether your compute node supports hardware acceleration for virtual machines
```
egrep -c '(vmx|svm)' /proc/cpuinfo
```

Edit the [libvirt] section in the /etc/nova/nova-compute.conf file as follows
```
vim /etc/nova/nova-compute.conf
```

Set this
```
[libvirt]
# ...
virt_type = qemu # Set this   
```

Restart the Compute service
```
systemctl restart nova-compute
``` 

List service components to verify successful launch and registration of each process
```
openstack compute service list
```

## Add the compute node to the cell database
Discover compute hosts
```
su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
``` 

List compute hosts
```
su -s /bin/sh -c "nova-manage cell_v2 list_hosts" nova
```     

Check the cells and placement API are working successfully and that other necessary prerequisites are in place
```
nova-status upgrade check
```