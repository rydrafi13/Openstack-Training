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
connection = mysql+pymysql://neutron:password@controller/neutron

[DEFAULT]
# ...
core_plugin = ml2
service_plugins = router

transport_url = rabbit://openstack:password@controller

auth_strategy = keystone

notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true


[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000/
auth_url = http://controller:5000/
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = password

[nova]
# ...
auth_url = http://controller:5000/
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
flat_networks = ext-net

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
physical_interface_mappings = provider:ens192

[vxlan]
enable_vxlan = true
local_ip = 10.0.0.8
l2_population = true

[securitygroup]
# ...
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

```
vim /etc/neutron/l3_agent.ini
```

```
[DEFAULT]
# ...
interface_driver = linuxbridge
```

```
vim /etc/neutron/dhcp_agent.ini
```

```
[DEFAULT]
# ...
interface_driver = linuxbridge
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
```

```
vim /etc/neutron/metadata_agent.ini
```

```
[DEFAULT]
# ...
nova_metadata_host = controller
metadata_proxy_shared_secret = rafi_secret
```

```
vim /etc/nova/nova.conf
```

```
# add this

[neutron]
# ...
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = Region-JKT
project_name = service
username = neutron
password = password
service_metadata_proxy = true
metadata_proxy_shared_secret = rafi_secret
```

```
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```

```
systemctl restart nova-{api,compute} 
```

```
systemctl restart neuton-{server,linuxbridge-agent,dhcp-agent,metadata-agent,l3-agent}
```