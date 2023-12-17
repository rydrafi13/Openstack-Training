# Keystone

## Setup Database
Before you install and configure the Identity service, you must create a database.
```
mysql -e "CREATE DATABASE keystone;"
mysql -e "GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'password';"
mysql -e "GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'password';"
mysql -e "flush privileges;"
```

Check user access database
```
mysql -e "SELECT User, Db, Host from mysql.db;"
```

## Install and Configure Keystone
Run the following command to install the packages
```
apt install keystone
```

Edit the /etc/keystone/keystone.conf file and complete the following actions
```
vim /etc/keystone/keystone.conf 
```

Set this
```
[cache]
memcache_servers = 10.0.0.8:11211 # Line 443

[database]
# ...
connection = mysql+pymysql://keystone:password@controller/keystone # Line 661

[token]
# ...
provider = fernet # Line 2642
```

Populate the Identity service database
```
su -s /bin/sh -c "keystone-manage db_sync" keystone
```

Initialize Fernet key repositories
```
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```

Bootstrap the Identity service
```
keystone-manage bootstrap --bootstrap-password password \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id JKT-01
```

## Configure apache HTTP server
Edit the /etc/apache2/apache2.conf file and configure the ServerName option to reference the controller node
```
vim /etc/apache2/apache2.conf
```

Set this
```
ServerName controller # line 70
```

Restart Service
```
systemctl restart apache2
```

## Create rc to connect via cli
Configure the administrative account by setting the proper environmental variables
```
vim admin-rc
```

Set this
```
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=password
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
export PS1='\u@\h \W(admin)\$ '
```

Load config
```
source admin-rc
```

## Manage keystone via CLI
The Identity service provides authentication services for each OpenStack service. The authentication service uses a combination of domains, projects, users, and roles.
```
# create project service
openstack project create --domain default --description "Service Project" service
```

Create domain, project, user, and roles
```
# create domain
openstack domain create --description "An Private Domain" private

# create project 
openstack project create --domain private --description "Personal Project" personal

# create user
openstack user create --domain private --password-prompt rafiryd

# add role admin to user on project 
openstack role add --project personal --user rafiryd member
```