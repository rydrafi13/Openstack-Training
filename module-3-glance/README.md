# Glance

## Setup Database
Before you install and configure the Image service, you must create a database, service credentials, and API endpoints.
```
mysql -e "CREATE DATABASE glance;"
mysql -e "GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'password';"
mysql -e "GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'password';"
mysql -e "flush privileges;"
```

Check user access database
```
mysql -e "SELECT User, Db, Host from mysql.db;"
```

## Setup endpoint glance
To create the service credentials, complete these steps
```
openstack user create --domain default --password-prompt glance
openstack role add --project service --user glance admin
openstack service create --name glance --description "OpenStack Image" image
```

Create the Image service API endpoints:
```
openstack endpoint create --region JKT-01 image public http://controller:9292
openstack endpoint create --region JKT-01 image internal http://controller:9292
openstack endpoint create --region JKT-01 image admin http://controller:9292
```

## Install and Configure Glance
Install the packages
```
apt install glance
```

Edit the /etc/glance/glance-api.conf file and complete the following actions
```
vim /etc/glance/glance-api.conf
```

Set this
```
[DEFAULT]
use_keystone_quotas = True

[database]
# ...
connection = mysql+pymysql://glance:password@controller/glance

[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = password

[paste_deploy]
# ...
flavor = keystone

[glance_store]
# ...
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
```

Populate the Image service database
```
su -s /bin/sh -c "glance-manage db_sync" glance    
```

Restart the Image services
```
systemctl restart glance-api
```

## Upload image to glance
Download the source image
```
wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
```

Upload the image to the Image service using the QCOW2 disk format, bare container format, and public visibility so all projects can access it
```
glance image-create --name "cirros" \
  --file cirros-0.4.0-x86_64-disk.img \
  --disk-format qcow2 --container-format bare \
  --visibility=public
```

Confirm upload of the image and validate attributes
```
glance image-list
```