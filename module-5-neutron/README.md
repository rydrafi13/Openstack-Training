# Neutron

## Setup Database
```
mysql -e "CREATE DATABASE neutron;"
mysql -e "GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'password';"
mysql -e "GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'password';"
mysql -e "flush privileges;"
```

```
mysql -e "SELECT User, Db, Host from mysql.db;"
```

## Setup endpoint
```
openstack user create --domain default --password-prompt neutron
openstack role add --project service --user neutron admin
openstack service create --name neutron --description "OpenStack Networking" network

openstack endpoint create --region Region-JKT network public http://controller:9696
openstack endpoint create --region Region-JKT network internal http://controller:9696
openstack endpoint create --region Region-JKT network admin http://controller:9696
```

## Install and Configuration Neutron
```
apt install neutron-server neutron-plugin-ml2 \
  neutron-linuxbridge-agent neutron-l3-agent neutron-dhcp-agent \
  neutron-metadata-agent
```

```
vim /etc/neutron/neutron.conf
```

```
[database]
# ...
connection = mysql+pymysql://neutron:password@10.0.0.8/neutron

[DEFAULT]
# ...
core_plugin = ml2
service_plugins = router

transport_url = rabbit://openstack:password@10.0.0.8

auth_strategy = keystone

notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true


[keystone_authtoken]
# ...
www_authenticate_uri = http://10.0.0.8:5000/
auth_url = http://10.0.0.8:5000/
memcached_servers = 10.0.0.8:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = password

[nova]
# ...
auth_url = http://10.0.0.8:5000/
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = Region-JKT
project_name = service
username = nova
password = password

[oslo_concurrency]
# ...
lock_path = /var/lib/neutron/tmp

[experimental]
linuxbridge = true

[agent]
root_helper = sudo /usr/bin/neutron-rootwrap /etc/neutron/rootwrap.conf

[privsep]
user = neutron
helper_command = sudo privsep-helper
```

```
vim /etc/neutron/plugins/ml2/ml2_conf.ini
```

```
[ml2]
# ...
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = linuxbridge,l2population
extension_drivers = port_security

[ml2_type_flat]
# ...
flat_networks = provider

[ml2_type_vxlan]
# ...
vni_ranges = 1:1000

[securitygroup]
# ...
enable_ipset = true
```

```
vim /etc/neutron/plugins/ml2/linuxbridge_agent.ini
```

```
[linux_bridge]
physical_interface_mappings = provider:PROVIDER_INTERFACE_NAME

[vxlan]
enable_vxlan = true
local_ip = OVERLAY_INTERFACE_IP_ADDRESS
l2_population = true

[securitygroup]
# ...
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

```
openstack --os-placement-api-version 1.2 resource class list --sort-column name
openstack --os-placement-api-version 1.6 trait list --sort-column name
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

openstack endpoint create --region Region-JKT compute public http://controller:8774/v2.1
openstack endpoint create --region Region-JKT compute internal http://controller:8774/v2.1
openstack endpoint create --region Region-JKT compute admin http://controller:8774/v2.1
```

## Install and Configuration Nova
```
apt install nova-api nova-conductor nova-novncproxy nova-scheduler python3-pip 
pip3 install oslo.cache
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
instances_path=/var/lib/nova/instances
my_ip = 10.0.0.8

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

## Install and Configuration Nova Compute
```
apt install nova-compute
```

```
vim /etc/nova/nova-compute.conf
```

```
[libvirt]
# ...
virt_type = qemu    
```

```
systemctl restart nova-compute
``` 

## Verify
```
su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
``` 