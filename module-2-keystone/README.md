# Keystone

## Setup Database
```
mysql -e "CREATE DATABASE keystone;"
mysql -e "GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'password';"
mysql -e "GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'password';"
mysql -e "flush privileges;"
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
## Create rc to connect via cli
## Create Project, and User
