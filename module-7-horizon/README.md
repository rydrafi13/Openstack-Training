# Horizon

## Install and Configure Horizon
Install the packages
```
apt install openstack-dashboard
```

Edit the /etc/openstack-dashboard/local_settings.py file and complete the following actions
```
vim /etc/openstack-dashboard/local_settings.py
```

Set this
```
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
        'LOCATION': '10.0.0.8:11211',
    },
}

SESSION_ENGINE = 'django.contrib.sessions.backends.cache'

OPENSTACK_HOST = "10.0.0.8"
OPENSTACK_KEYSTONE_URL = "http://%s:5000/identity/v3" % OPENSTACK_HOST

TIME_ZONE = "Asia/Jakarta"

OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"
OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 3,
}
```

Reload the web server configuration
```
systemctl reload apache2.service
```

Access Horizon
```
# on browser
http://10.0.0.8/horizon/

# login
user: admin
password: password
domain: default
```