# Openstack Admninistration

## Manage keystone via CLI
Create project, user, and roles
```
# create project 
openstack project create --domain default --description "Personal Project" personal

# create user
openstack user create --domain default --password-prompt rafiryd

# add role admin to user on project 
openstack role add --project personal --user rafiryd member
```

## Upload image to glance
Download the source image
```
curl -LO https://cloud-images.ubuntu.com/bionic/current/bionic-server-cloudimg-amd64.img
```

Upload the image to the Image service using the QCOW2 disk format, bare container format, and public visibility so all projects can access it
```
glance image-create --name "ubuntu-18" \
  --file bionic-server-cloudimg-amd64.img \
  --disk-format qcow2 --container-format bare \
  --visibility=public
```

Confirm upload of the image and validate attributes
```
glance image-list
```

## Launch an instance
### Create Virtual Network
Public Network
```
openstack network create  --share --external \
  --provider-physical-network physnet1 \
  --provider-network-type flat public-net
```

Public Subnet
```
openstack subnet create --network public-net \
  --allocation-pool start=10.20.30.x,end=10.20.30.x \
  --dns-nameserver 1.1.1.1 --gateway 10.20.30.254 \
  --subnet-range 10.20.30.0/24 public-subnet
```

Private Network
```
openstack network create private-net
```

Private Subnet
```
openstack subnet create --network private-net \
  --dns-nameserver 1.1.1.1 --gateway 10.50.100.254 \
  --subnet-range 10.50.100.0/24 private-subnet
```

List network and subnet
```
# network
openstack network list

# subnet
openstack subnet list
```

### Create Flavor
Create m1.nano flavor
```
openstack flavor create --id 0 --vcpus 1 --ram 1024 --disk 10 m1.nano
```

List flavor
```
openstack flavor list
```

### Generate Keypair
Generate a key pair and add a public key
```
# generate
ssh-keygen -q -N ""

# add pubkey
openstack keypair create --public-key ~/.ssh/id_rsa.pub host-key
```

List keypair
```
openstack keypair list
```

### Add security group rules
Create security group
```
openstack security group create SG_Rafi
```

Add rules to the default security group Permit ICMP (ping)
```
openstack security group rule create --proto icmp SG_Rafi
```

Add rules to the default security group Permit secure shell (SSH)
```
openstack security group rule create --proto tcp --dst-port 22 SG_Rafi
```

List security group
```
openstack security group list
```

List rule security group
```
openstack security group rule list
```

### Launch an instance
Launch Public-VM
```
openstack server create --flavor m1.nano --image cirros \
  --nic net-id=PUBLIC_NET_ID --security-group SG_Rafi \
  --key-name aio-key Public-VM
```

Launch Private-VM
```
openstack server create --flavor m1.nano --image cirros \
  --nic net-id=PRIVATE_NET_ID --security-group SG_Rafi \
  --key-name aio-key Private-VM 
```

Check the status of your instance
```
openstack server list
```

Obtain a Virtual Network Computing (VNC) session URL for your instance and access it from a web browser
```
openstack console url show VM
```

cloud-init sample
```
#cloud-config

users:
  - name: rafiryd
    passwd: $6$rounds=10000$SALTSTRING$ENCRYPTEDPASSWORD
    sudo: ALL=(ALL) NOPASSWD:ALL
    groups: sudo
    shell: /bin/bash
    ssh_pwauth: true
```

## Manage Volume via CLI
Create a 1 GB volume
```
openstack volume create --size 1 volume1
```

List volume
```
openstack volume list
```

Attach a volume to an instance
```
openstack server add volume Public-VM volume1
```

## Manage Stack
Create Stack Template
```
vim stack1.yml
```

Set this
```
heat_template_version: 2015-10-15

resources:
  web_secgroup:
    type: OS::Neutron::SecurityGroup
    properties:
      rules:
        - protocol: icmp
          remote_ip_prefix: 0.0.0.0/0
        - protocol: tcp
          remote_ip_prefix: 0.0.0.0/0
          port_range_min: 22
          port_range_max: 22

  volume:
    type: OS::Cinder::Volume
    properties:
      size: 2


  instance_port:
    type: OS::Neutron::Port
    properties:
      network: public-net
      security_groups:
        - default
        - { get_resource: web_secgroup }
      fixed_ips:
        - subnet_id: public-subnet

  instance:
    type: OS::Nova::Server
    properties:
      flavor: m1.small
      image: cirros
      networks:
        - port: { get_resource: instance_port }

  volume_attachment:
    type: OS::Cinder::VolumeAttachment
    properties:
      volume_id: { get_resource: volume }
      instance_uuid: { get_resource: instance }
```

Create stack
```
openstack stack create -t stack1.yml stack1
```
