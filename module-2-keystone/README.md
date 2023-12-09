# Keystone

## Setup Database
```
mysql -e "CREATE DATABASE keystone;"
mysql -e "GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'password';"
mysql -e "GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'password';"
mysql -e "flush privileges;"
```

```
mysql -e "SELECT User, Db, Host from mysql.db;"
```

## Install and Configure Keystone
```
apt install keystone
```

```
/etc/keystone/keystone.conf 
```

```
[cache]
memcache_servers = 10.0.0.8:11211

[database]
# ...
connection = mysql+pymysql://keystone:password@controller/keystone

[token]
# ...
provider = fernet
```

```
su -s /bin/sh -c "keystone-manage db_sync" keystone
```

```
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```

```
keystone-manage bootstrap --bootstrap-password password \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id Region-JKT
```

## Configure apache HTTP server
```
vim /etc/apache2/apache2.conf
```

```
ServerName controller
```

```
systemctl restart apache2
```

## Create rc to connect via cli
```
vim admin-rc
```

```
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=password
export OS_AUTH_URL=http://controller    :5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
export PS1='\u@\h \W(admin)\$ '
```

```
source admin-rc
```

## Create Project, and User
```
# create project service
openstack project create --domain default --description "Service Project" service

# create own project 
openstack project create --domain default --description "Personal Project" personal
openstack user create --domain default --password-prompt rafiryd
openstack role create client
openstack role add --project personal --user rafiryd client
```
