# Kolla Ansible

## Prepare Installation
### Set Hostname
Set hostname node
```
hostnamectl set-hostname controller/compute/storage
```

### Set Timezone
Set timezone to Asia/Jakarta
```
# set timezone to WIB
timedatectl set-timezone Asia/Jakarta

# check timezone
date
```

### Set Static IP
Edit netplan config
```
vim /etc/netplan/00-installer-config.yaml
```

Set static ip address
```
network:
  version: 2
  renderer: networkd
  ethernets:
    ens192:
      dhcp4: false
    ens160:
      dhcp4: false
      addresses:
        - 10.0.0.x/24
      routes:
        - to: default
          via: 10.0.0.1
      nameservers:
          search: [google.com]
          addresses: [1.1.1.1, 8.8.8.8]
```

Load config
```
netplan apply
```

### Update ,Upgrade, and Cleaning Packages
Update repository & Update packages
```
sudo apt update -y && apt upgrade -y
```

Cleaning package
```
sudo apt autoremove -y && apt clean all
```

### Set /etc/host
Edit /etc/host
```
sudo vim /etc/hosts
```

Set hostname to host
```
# openstack environtment
10.0.0.x controller
10.0.0.x compute
10.0.0.x storage
```

## Prepare Environtment
### Install SSH Pass
Install sshpass
```
sudo apt install sshpass
```

### Set SSH Config
Set ssh config
```
sudo vim /etc/ssh_config
```

Set this
```
StrictHostKeyChecking no
```

### Generate SSH Key
Generate SSH Key based RSA
```
ssh-keygen -t rsa
```

### Copy SSH Key to all node
Create list hosts
```
vim hosts
```

Set this
```
controller01
controller02
controller03
compute01
compute02
storage01
```

Create password login for user
```
echo "ubuntu" > pass
```

Copy SSH to all node on list with credentials password
```
for hostname in `cat hosts`; do sshpass -f pass ssh-copy-id ubuntu@${hostname}; done
```

### Setup Volume on storage node
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

### Install Python Dependecies
Install Python build dependencies
```
sudo apt install python3-dev libffi-dev gcc libssl-dev
```

### Create Python venv
Install the virtual environment dependencies
```
sudo apt install python3-venv
```

Create a directory
```
mkdir kolla
```

Create a virtual environment and activate it
```
python3 -m venv kolla/venv
source kolla/venv/bin/activate
```

Ensure the latest version of pip is installed
```
pip install -U pip
```

### Install Ansible
Install Ansible. Kolla Ansible requires at least Ansible 4 and supports up to 5
```
pip install 'ansible>=4,<6'
```

## Installation Kolla Ansible
### Install Kolla-Ansible
Install kolla-ansible and its dependencies using pip
```
pip install git+https://opendev.org/openstack/kolla-ansible@stable/zed
```

Create the /etc/kolla directory
```
sudo mkdir -p /etc/kolla
sudo chown $USER:$USER /etc/kolla
```

Copy the configuration files to /etc/kolla directory. kolla-ansible holds the configuration files ( globals.yml and passwords.yml) in etc/kolla
```
cp -r kolla/venv/share/kolla-ansible/etc_examples/kolla/* /etc/kolla/ 
```

Copy the inventory files to the current directory. kolla-ansible holds inventory files ( all-in-one and multinode) in the ansible/inventory directory
```
cp kolla/venv/share/kolla-ansible/ansible/inventory/* .
```

### Install Ansible Galaxy requirements
Install Ansible Galaxy dependencies (Yoga release onwards)
```
kolla-ansible install-deps
```

Create directory ansible
```
sudo mkdir -p /etc/ansible
sudo chown $USER:$USER /etc/ansible
```

Create ansible config
```
vim /etc/ansible/ansible.cfg
```

Set this
```
[defaults]
host_key_checking=False
pipelining=True
forks=100
```

### Prepare initial configuration
Edit the first section of multinode with connection details of your environment, for example
```
vim multinode
```

Set this
```
[control]
controller01 ansible_become_password=ubuntu ansible_become=true
controller02 ansible_become_password=ubuntu ansible_become=true

[network]
controller01 ansible_become_password=ubuntu ansible_become=true
controller02 ansible_become_password=ubuntu ansible_become=true

[compute]
compute01 ansible_become_password=ubuntu ansible_become=true

[monitoring]
controller01 ansible_become_password=ubuntu ansible_become=true

[storage]
storage01 ansible_become_password=ubuntu ansible_become=true

[deployment]
localhost       ansible_connection=local become=true
```

Check whether the configuration of inventory is correct or not, run
```
ansible -i multinode all -m ping
```

Kolla passwords
```
kolla-genpwd
```

globals.yml is the main configuration file for Kolla Ansible. There are a few options that are required to deploy Kolla Ansible
```
vim /etc/kolla/globals.yml
```

Set this
```
kolla_base_distro: "ubuntu"
openstack_release: "zed"
kolla_internal_vip_address: "10.0.0.x" # Range IP Available
network_interface: "ens160"
neutron_external_interface: "ens192"
openstack_region_name: "JKT-01"
enable_cinder: "yes"
enable_cinder_backup: "no"  
enable_cinder_backend_lvm: "yes"
enable_etcd: "yes"
enable_horizon: "{{ enable_openstack_core | bool }}"
```

Check options globals.yml
```
cat /etc/kolla/globals.yml | grep -v "#" |  tr -s [:space:]
```

### Deployment Kolla-Ansible
Bootstrap servers with kolla deploy dependencies
```
kolla-ansible -i ./multinode bootstrap-servers
```

Do pre-deployment checks for hosts
```
kolla-ansible -i ./multinode prechecks
```

Do download image locally
```
kolla-ansible -i ./multinode pull
```

Finally proceed to actual OpenStack deployment
```
kolla-ansible -i ./multinode deploy
```

### Post Deployment Kolla-Ansible
Install the OpenStack CLI client
```
pip install python-openstackclient -c https://releases.openstack.org/constraints/upper/zed
```

OpenStack requires a clouds.yaml file where credentials for the admin user are set. To generate this file
```
kolla-ansible post-deploy
```

Check credentials
```
cat /etc/kolla/admin-openrc.sh
```

Login Horizon
```
http://10.0.0.x/
```

### Add node controller
Edit the first section of multinode with connection details of your environment
```
vim multinode 
```

Set this
```
[control]
controller01 ansible_become_password=ubuntu ansible_become=true
controller02 ansible_become_password=ubuntu ansible_become=true
controller03 ansible_become_password=ubuntu ansible_become=true

[network]
controller01 ansible_become_password=ubuntu ansible_become=true
controller02 ansible_become_password=ubuntu ansible_become=true
controller03 ansible_become_password=ubuntu ansible_become=true
```

Check whether the configuration of inventory is correct or not, run
```
ansible -i multinode all -m ping
```

The bootstrap-servers command, can be used to prepare the new hosts that are being added to the system. Be aware of the potential issues with running bootstrap-servers on an existing system
```
kolla-ansible -i multinode bootstrap-servers --limit controller03
```

Do pre-deployment checks for hosts
```
kolla-ansible -i multinode prechecks --limit controller03
```

Pull down container images to the new hosts. The --limit argument may be used and only needs to include the new hosts
```
kolla-ansible -i multinode pull --limit controller03
```

Deploy containers to the new hosts. If using a --limit argument, ensure that all controllers are included, e.g. via --limit control
```
kolla-ansible -i multinode deploy --limit control
```

### Add node compute
Edit the first section of multinode with connection details of your environment
```
vim multinode 
```

Set this
```
[compute]
compute01 ansible_become_password=ubuntu ansible_become=true
compute02 ansible_become_password=ubuntu ansible_become=true
```

Check whether the configuration of inventory is correct or not, run
```
ansible -i multinode all -m ping
```

The bootstrap-servers command, can be used to prepare the new hosts that are being added to the system. Be aware of the potential issues with running bootstrap-servers on an existing system
```
kolla-ansible -i multinode bootstrap-servers --limit compute02
```

Do pre-deployment checks for hosts
```
kolla-ansible -i multinode prechecks --limit compute02
```

Pull down container images to the new hosts. The --limit argument may be used and only needs to include the new hosts
```
kolla-ansible -i multinode pull --limit compute02
```

Deploy containers on the new hosts. The --limit argument may be used and only needs to include the new hosts
```
kolla-ansible -i multinode deploy --limit compute02
```

### Add addtional service
globals.yml is the main configuration file for Kolla Ansible. There are a few options that are required to deploy Kolla Ansible
```
vim /etc/kolla/globals.yml
```

Set this
```
kolla_base_distro: "ubuntu"
openstack_release: "zed"
kolla_internal_vip_address: "10.0.0.x" # Range IP Available
network_interface: "ens160"
neutron_external_interface: "ens192"
openstack_region_name: "JKT-01"
enable_cinder: "yes"
enable_cinder_backup: "no"  
enable_cinder_backend_lvm: "yes"
enable_etcd: "yes"
enable_horizon: "{{ enable_openstack_core | bool }}"
enable_horizon_masakari: "{{ enable_masakari | bool }}"
enable_horizon_watcher: "{{ enable_watcher | bool }}"
enable_masakari: "yes"
enable_watcher: "yes"
```

Check options globals.yml
```
cat /etc/kolla/globals.yml | grep -v "#" |  tr -s [:space:]
```

The bootstrap-servers command, can be used to prepare the new hosts that are being added to the system. Be aware of the potential issues with running bootstrap-servers on an existing system
```
kolla-ansible -i multinode bootstrap-servers --tags watcher,masakari
```

Do pre-deployment checks for hosts
```
kolla-ansible -i multinode prechecks --tags watcher,masakari
```

Pull down container images to the new hosts. The --limit argument may be used and only needs to include the new hosts
```
kolla-ansible -i multinode pull --tags watcher,masakari
```

Deploy containers on the new hosts. The --limit argument may be used and only needs to include the new hosts
```
kolla-ansible -i multinode deploy --tags watcher,masakari
```