# Contrail 5.0 + OpenStack kolla ALL In One Install

## 0. Rqeuirement

Vagrant Box: centos/7

CentOS: CentOS Linux release 7.4.1708 (Core)

Kernel: 3.10.0-862.3.2.el7.x86_64 #1 SMP Mon May 21 23:36:36 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux

Network Interface: 1NIC

KVM Host: Enable Nested

contrail-ansible-deployer: commit 68549ff687c5a0db443bc737a17f02b3eeb54844

contrail-kolla-ansible: commit 4039df48af4df04d0afb3b0c940356c7794a88f7

CONTAINER_REGISTRY: opencontrailnightly

CONTRAIL_VERSION: ocata-master-117

## 1. Create VM using by Vagrant

### 1.1. Create Vagrant file & Directory
Target: **Host**

```
$ cd ~
$ mkdir c01
$ cd c01

$ cat <<EOF > Vagrantfile
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "centos/7"
  config.vm.define :k1 do |k1|
  end
    config.vm.provider :libvirt do |lv|
      lv.uri = 'qemu+unix:///system'
      lv.cpus = 4
      lv.memory = 65536
      lv.boot 'hd'
      lv.management_network_name = "vag-mgmt01"
      lv.management_network_autostart = "ture"
      lv.management_network_address = "192.168.120.0/24"
      lv.management_network_autostart = "ture"
    end
end
EOF
```

### 1.2. Deploy VM & Login
```
$ vagrant up
$ vagrant ssh
```

# 2. Prepare deploy
Target: **VM**
```
$ sudo su -
```

### 2.1. Enable Password authentication
```
$ sudo sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config
$ sudo systemctl restart sshd
$ sudo passwd root
```

### 2.2. Install require packages
```
$ sudo yum install -y vim ntp epel-release git ansible net-tools
```

### 2.3. Start and enable ntpd
```
$ sudo systemctl restart ntpd
$ sudo systemctl enable ntpd
```

### 2.4. Increase disk size
*The CentOS Disk size defaults to 40 GB, however AIO Requirements will require 300 GB. So we need to increase the Disk size*

Target: **VM**
```
$ sudo shutdown -h now
```

Target: **Host**
```
cd /var/lib/libvirt/images/
sudo qemu-img resize c01_k1.img +250G

cd ~/c01
vagrant up
vagrant ssh
```

Target: **VM**
```
sudo fdisk /dev/vda
======================================================================
Welcome to fdisk (util-linux 2.23.2).

Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): d
Partition number (1-3, default 3):
Partition 3 is deleted

Command (m for help): n
Partition type:
   p   primary (2 primary, 0 extended, 2 free)
   e   extended
Select (default p):
Using default response p
Partition number (3,4, default 3):
First sector (2101248-610271231, default 2101248):
Using default value 2101248
Last sector, +sectors or +size{K,M,G} (2101248-610271231, default 610271231):
Using default value 610271231
Partition 3 of type Linux and of size 290 GiB is set

Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.

WARNING: Re-reading the partition table failed with error 16: Device or resource busy.
The kernel still uses the old table. The new table will be used at
the next reboot or after you run partprobe(8) or kpartx(8)
Syncing disks.
======================================================================

sudo reboot
```

Target: **Host**
```
vagrant ssh
```

Target: **VM**
```
$ sudo su -
$ sudo pvresize /dev/vda3
$ sudo lvextend -L +250G /dev/mapper/VolGroup00-LogVol00
$ sudo xfs_growfs /dev/mapper/VolGroup00-LogVol00
```
### 2.5. Configure /etc/hosts

**NW information in my environment**
```
======================================================================
[root@localhost ~]# ip -o a
1: lo    inet 127.0.0.1/8 scope host lo\       valid_lft forever preferred_lft forever
1: lo    inet6 ::1/128 scope host \       valid_lft forever preferred_lft forever
2: eth0    inet 192.168.120.226/24 brd 192.168.120.255 scope global eth0\       valid_lft forever preferred_lft forever
2: eth0    inet6 fe80::5054:ff:fef1:5155/64 scope link \       valid_lft forever preferred_lft forever
======================================================================
```

### 2.6. Update Kernel

Current require kernel version at May 30.

Linux localhost.localdomain 3.10.0-862.3.2.el7.x86_64 #1 SMP Mon May 21 23:36:36 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux

```
$ yum -y update kernel
$ sudo reboot
```

### 2.7 Clone repository 
```
$ cd ~/
$ git clone http://github.com/Juniper/contrail-ansible-deployer
$ cd contrail-ansible-deployer/
```

### 2.8. Apply patch
```
sed -i 's/roles.item /roles[item] /g' playbooks/roles/create_openstack_config/tasks/host_params.yml
```

***[Patch information](https://github.com/Juniper/contrail-ansible-deployer/pull/14)***

### 2.9. Configuration instances.yaml

**NW information in my environment (before deploy)**
```
======================================================================
[root@localhost ~]# ip -o a
1: lo    inet 127.0.0.1/8 scope host lo\       valid_lft forever preferred_lft forever
1: lo    inet6 ::1/128 scope host \       valid_lft forever preferred_lft forever
2: eth0    inet 192.168.120.226/24 brd 192.168.120.255 scope global eth0\       valid_lft forever preferred_lft forever
2: eth0    inet6 fe80::5054:ff:fef1:5155/64 scope link \       valid_lft forever preferred_lft forever
======================================================================
```

**NW information in my environment (after deploy)**
```
======================================================================
[root@localhost ~]# ip -o a
1: lo    inet 127.0.0.1/8 scope host lo\       valid_lft forever preferred_lft forever
1: lo    inet6 ::1/128 scope host \       valid_lft forever preferred_lft forever
2: eth0    inet6 fe80::5054:ff:fef1:5155/64 scope link \       valid_lft forever preferred_lft forever
8: vhost0    inet 192.168.120.226/24 brd 10.1.0.255 scope global vhost0\       valid_lft forever preferred_lft forever
8: vhost0    inet6 fe80::5054:ff:fedb:bf29/64 scope link \       valid_lft forever preferred_lft forever
9: docker0    inet 172.17.0.1/16 scope global docker0\       valid_lft forever preferred_lft forever
======================================================================
```

```
$ vim contrail-ansible-deployer/config/instances.yaml

provider_config:
  bms:
    ssh_pwd: lab
    ssh_user: root
    ntpserver: 210.173.160.27
    domainsuffix: localdomain
#    ssh_public_key: /home/centos/.ssh/id_rsa.pub  # Optional. Not needed if ssh password is used.
#    ssh_private_key: /home/centos/.ssh/id_rsa     # Optional. Not needed if ssh password is used.
instances:
  bms1:
    provider: bms
    ip: 192.168.120.226
    roles:
      config_database:
      config:
      control:
      analytics_database:
      analytics:
      webui:
      openstack_control:
      openstack_network:
        network_interface: eth0
      openstack_storage:
      openstack_monitoring:
      vrouter:
        VROUTER_GATEWAY: 192.168.120.1
        PHYSICAL_INTERFACE: eth0
      openstack_compute:
kolla_config:
  customize:
    nova.conf: |
      [libvirt]
      virt_type=kvm
      cpu_mode=host-passthrough
  kolla_globals:
    openstack_release: ocata
    enable_haproxy: no
    enable_swift: no
    enable_barbican: no
    enable_heat: no
    enable_ironic: no
    kolla_internal_vip_interface: eth0
    kolla_internal_vip_address: 192.168.120.226
    # heat/templates/heat.conf.j2:api_server = {{ contrail_api_interface_address }}
    contrail_api_interface_address: 192.168.120.226
    # neutron/templates/ContrailPlugin.ini.j2:api_server_ip = {{ opencontrail_api_server_ip }}
    opencontrail_api_server_ip: 192.168.120.226
  kolla_passwords:
    keystone_admin_password: lab
global_configuration:
  CONTAINER_REGISTRY: opencontrailnightly
#  REGISTRY_PRIVATE_INSECURE: True
#  CONTAINER_REGISTRY_USERNAME: YourRegistryUser
#  CONTAINER_REGISTRY_PASSWORD: YourRegistryPassword
contrail_configuration:
  CONTRAIL_VERSION: ocata-master-117
  UPGRADE_KERNEL: true
  CLOUD_ORCHESTRATOR: openstack
  CONTROLLER_NODES: 192.168.120.226
  CONTROL_NODES: 192.168.120.226
  ANALYTICSDB_NODES: 192.168.120.226
  WEBUI_NODES: 192.168.120.226
  ANALYTICS_NODES: 192.168.120.226
  CONFIGDB_NODES: 192.168.120.226
  CONFIG_NODES: 192.168.120.226
  RABBITMQ_NODE_PORT: 5673
  AUTH_MODE: keystone
  KEYSTONE_AUTH_URL_VERSION: /v3
  KEYSTONE_AUTH_HOST: 192.168.120.226

```

### 2.10. Configure instance
```
$ cd contrail-ansible-deployer
$ ansible-playbook -i inventory/ playbooks/configure_instances.yml
```

### 2.11. Apply patch
```
$ sed -i 's/use_neutron = True//g' ~/contrail-kolla-ansible/ansible/roles/nova/templates/nova.conf.j2
```

***[Patch information](https://bugs.launchpad.net/kolla-ansible/+bug/1651665)***

### 2.11.1 Deploy OpenStack
```
$ ansible-playbook -i inventory/ -e orchestrator=openstack playbooks/install_openstack.yml
```

### 2.11.2 Deploy Contrail
```
$ ansible-playbook -i inventory/ -e orchestrator=openstack playbooks/install_contrail.yml
```

### 2.12. Check contrail-status

```
[root@localhost ~]# contrail-status
Pod        Service         Original Name                          State    Status
analytics  alarm-gen       contrail-analytics-alarm-gen           running  Up 18 minutes
analytics  api             contrail-analytics-api                 running  Up 20 minutes
analytics  collector       contrail-analytics-collector           running  Up 20 minutes
analytics  nodemgr         contrail-nodemgr                       running  Up 20 minutes
analytics  query-engine    contrail-analytics-query-engine        running  Up 20 minutes
config     api             contrail-controller-config-api         running  Up 20 minutes
config     cassandra       contrail-external-cassandra            running  Up 20 minutes
config     device-manager  contrail-controller-config-devicemgr   running  Up 20 minutes
config     nodemgr         contrail-nodemgr                       running  Up 20 minutes
config     rabbitmq        contrail-external-rabbitmq             running  Up 20 minutes
config     schema          contrail-controller-config-schema      running  Up 20 minutes
config     svc-monitor     contrail-controller-config-svcmonitor  running  Up 20 minutes
config     zookeeper       contrail-external-zookeeper            running  Up 20 minutes
control    control         contrail-controller-control-control    running  Up 20 minutes
control    dns             contrail-controller-control-dns        running  Up 20 minutes
control    named           contrail-controller-control-named      running  Up 20 minutes
control    nodemgr         contrail-nodemgr                       running  Up 20 minutes
database   cassandra       contrail-external-cassandra            running  Up 20 minutes
database   kafka           contrail-external-kafka                running  Up 20 minutes
database   nodemgr         contrail-nodemgr                       running  Up 20 minutes
database   zookeeper       contrail-external-zookeeper            running  Up 20 minutes
vrouter    agent           contrail-vrouter-agent                 running  Up 20 minutes
vrouter    nodemgr         contrail-nodemgr                       running  Up 20 minutes
webui      job             contrail-controller-webui-job          running  Up 20 minutes
webui      web             contrail-controller-webui-web          running  Up 20 minutes

vrouter kernel module is PRESENT
== Contrail control ==
control: active
nodemgr: active
named: active
dns: active

== Contrail database ==
kafka: active
nodemgr: active
zookeeper: active
cassandra: active

== Contrail analytics ==
nodemgr: active
api: active
collector: active
query-engine: active
alarm-gen: active

== Contrail webui ==
web: active
job: active

== Contrail vrouter ==
nodemgr: active
agent: active

== Contrail config ==
api: active
zookeeper: active
svc-monitor: active
nodemgr: active
device-manager: active
cassandra: active
rabbitmq: active
schema: active
```

:)

# Tips 

### How to start openstack

```
[root@localhost contrail-kolla-ansible]# pip install python-openstackclient

-- snip --

Successfully installed PrettyTable-0.7.2 appdirs-1.4.3 asn1crypto-0.24.0 cffi-1.11.5 cliff-2.12.0 cmd2-0.8.7 contextlib2-0.5.5 cryptography-2.2.2 deprecation-2.0.2 dogpile.cache-0.6.5 futures-3.2.0 jsonpatch-1.23 jsonpointer-2.0 keystoneauth1-3.7.0 msgpack-0.5.6 munch-2.3.2 openstacksdk-0.13.0 os-client-config-1.31.1 os-service-types-1.2.0 osc-lib-1.10.0 oslo.serialization-2.25.0 packaging-17.1 pyOpenSSL-18.0.0 pyperclip-1.6.1 python-cinderclient-3.5.0 python-glanceclient-2.11.0 python-keystoneclient-3.16.0 python-novaclient-10.2.0 python-openstackclient-3.15.0 requestsexceptions-1.4.0 subprocess32-3.5.1 unicodecsv-0.14.1 warlock-1.3.0 wcwidth-0.1.7
You are using pip version 8.1.2, however version 10.0.1 is available.
You should consider upgrading via the 'pip install --upgrade pip' command.

[root@localhost contrail-kolla-ansible]# source /etc/kolla/admin-openrc.sh

[root@localhost contrail-kolla-ansible]# sudo yum -y install wget

-- snip --

  Installing : wget-1.14-15.el7_4.1.x86_64                                                                                                                                                                                                                 1/1
  Verifying  : wget-1.14-15.el7_4.1.x86_64                                                                                                                                                                                                                 1/1

Installed:
  wget.x86_64 0:1.14-15.el7_4.1

Complete!

[root@localhost contrail-kolla-ansible]# wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
--2018-05-30 18:12:45--  http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
Resolving download.cirros-cloud.net (download.cirros-cloud.net)... 64.90.42.85, 2607:f298:6:a036::bd6:a72a
Connecting to download.cirros-cloud.net (download.cirros-cloud.net)|64.90.42.85|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 12716032 (12M) [text/plain]
Saving to: ‚Äòcirros-0.4.0-x86_64-disk.img‚Äô

100%[=====================================================================================================================================================================================================================>] 12,716,032  2.65MB/s   in 12s

2018-05-30 18:12:57 (1.02 MB/s) - ‚Äòcirros-0.4.0-x86_64-disk.img‚Äô saved [12716032/12716032]


[root@localhost contrail-kolla-ansible]# openstack image create cirros2 --disk-format qcow2 --public --container-format bare --file cirros-0.4.0-x86_64-disk.img
+------------------+------------------------------------------------------+
| Field            | Value                                                |
+------------------+------------------------------------------------------+
| checksum         | 443b7623e27ecf03dc9e01ee93f67afe                     |
| container_format | bare                                                 |
| created_at       | 2018-05-30T18:13:02Z                                 |
| disk_format      | qcow2                                                |
| file             | /v2/images/480d6322-6089-4e2f-aaa2-5810f5e3b0db/file |
| id               | 480d6322-6089-4e2f-aaa2-5810f5e3b0db                 |
| min_disk         | 0                                                    |
| min_ram          | 0                                                    |
| name             | cirros2                                              |
| owner            | 948b76c981d248beb070aa399ec4e40f                     |
| protected        | False                                                |
| schema           | /v2/schemas/image                                    |
| size             | 12716032                                             |
| status           | active                                               |
| tags             |                                                      |
| updated_at       | 2018-05-30T18:13:03Z                                 |
| virtual_size     | None                                                 |
| visibility       | public                                               |
+------------------+------------------------------------------------------+

[root@localhost contrail-kolla-ansible]# openstack network create testvn
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | UP                                   |
| availability_zone_hints   | None                                 |
| availability_zones        | None                                 |
| created_at                | None                                 |
| description               |                                      |
| dns_domain                | None                                 |
| id                        | 2299f8d8-4320-4598-b3ff-05c9c5216d45 |
| ipv4_address_scope        | None                                 |
| ipv6_address_scope        | None                                 |
| is_default                | None                                 |
| is_vlan_transparent       | None                                 |
| mtu                       | None                                 |
| name                      | testvn                               |
| port_security_enabled     | True                                 |
| project_id                | 948b76c981d248beb070aa399ec4e40f     |
| provider:network_type     | None                                 |
| provider:physical_network | None                                 |
| provider:segmentation_id  | None                                 |
| qos_policy_id             | None                                 |
| revision_number           | None                                 |
| router:external           | Internal                             |
| segments                  | None                                 |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tags                      |                                      |
| updated_at                | None                                 |
+---------------------------+--------------------------------------+

[root@localhost contrail-kolla-ansible]# openstack subnet create --subnet-range 192.168.100.0/24 --network testvn subnet1
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| allocation_pools  | 192.168.100.2-192.168.100.254        |
| cidr              | 192.168.100.0/24                     |
| created_at        | 2018-05-30T18:13:12.704717           |
| description       | None                                 |
| dns_nameservers   |                                      |
| enable_dhcp       | True                                 |
| gateway_ip        | 192.168.100.1                        |
| host_routes       |                                      |
| id                | 4e5c3056-5fc1-42c4-9887-38acd96af7df |
| ip_version        | 4                                    |
| ipv6_address_mode | None                                 |
| ipv6_ra_mode      | None                                 |
| name              | subnet1                              |
| network_id        | 2299f8d8-4320-4598-b3ff-05c9c5216d45 |
| project_id        | 948b76c981d248beb070aa399ec4e40f     |
| revision_number   | None                                 |
| segment_id        | None                                 |
| service_types     | None                                 |
| subnetpool_id     | None                                 |
| tags              |                                      |
| updated_at        | 2018-05-30T18:13:12.704717           |
+-------------------+--------------------------------------+

[root@localhost contrail-kolla-ansible]# openstack flavor create --ram 512 --disk 1 --vcpus 1 m1.tiny

+----------------------------+--------------------------------------+
| Field                      | Value                                |
+----------------------------+--------------------------------------+
| OS-FLV-DISABLED:disabled   | False                                |
| OS-FLV-EXT-DATA:ephemeral  | 0                                    |
| disk                       | 1                                    |
| id                         | 7030eb41-1cd1-499d-b23f-c1e5bce6db3b |
| name                       | m1.tiny                              |
| os-flavor-access:is_public | True                                 |
| properties                 |                                      |
| ram                        | 512                                  |
| rxtx_factor                | 1.0                                  |
| swap                       |                                      |
| vcpus                      | 1                                    |
+----------------------------+--------------------------------------+

[root@localhost contrail-kolla-ansible]# NET_ID=`openstack network list | grep testvn | awk -F '|' '{print $2}' | tr -d ' '`

[root@localhost contrail-kolla-ansible]# openstack server create --flavor m1.tiny --image cirros2 --nic net-id=${NET_ID} test_vm1
+-------------------------------------+------------------------------------------------+
| Field                               | Value                                          |
+-------------------------------------+------------------------------------------------+
| OS-DCF:diskConfig                   | MANUAL                                         |
| OS-EXT-AZ:availability_zone         |                                                |
| OS-EXT-SRV-ATTR:host                | None                                           |
| OS-EXT-SRV-ATTR:hypervisor_hostname | None                                           |
| OS-EXT-SRV-ATTR:instance_name       |                                                |
| OS-EXT-STS:power_state              | NOSTATE                                        |
| OS-EXT-STS:task_state               | scheduling                                     |
| OS-EXT-STS:vm_state                 | building                                       |
| OS-SRV-USG:launched_at              | None                                           |
| OS-SRV-USG:terminated_at            | None                                           |
| accessIPv4                          |                                                |
| accessIPv6                          |                                                |
| addresses                           |                                                |
| adminPass                           | 4c62J5smK3nJ                                   |
| config_drive                        |                                                |
| created                             | 2018-05-30T18:13:33Z                           |
| flavor                              | m1.tiny (7030eb41-1cd1-499d-b23f-c1e5bce6db3b) |
| hostId                              |                                                |
| id                                  | b01ac0d8-64ce-4e0b-8325-61acc115fa3b           |
| image                               | cirros2 (480d6322-6089-4e2f-aaa2-5810f5e3b0db) |
| key_name                            | None                                           |
| name                                | test_vm1                                       |
| progress                            | 0                                              |
| project_id                          | 948b76c981d248beb070aa399ec4e40f               |
| properties                          |                                                |
| security_groups                     | name='default'                                 |
| status                              | BUILD                                          |
| updated                             | 2018-05-30T18:13:33Z                           |
| user_id                             | fc353d7393644069a0abaad10e508a00               |
| volumes_attached                    |                                                |
+-------------------------------------+------------------------------------------------+

[root@localhost contrail-kolla-ansible]# openstack server create --flavor m1.tiny --image cirros2 --nic net-id=${NET_ID} test_vm2
+-------------------------------------+------------------------------------------------+
| Field                               | Value                                          |
+-------------------------------------+------------------------------------------------+
| OS-DCF:diskConfig                   | MANUAL                                         |
| OS-EXT-AZ:availability_zone         |                                                |
| OS-EXT-SRV-ATTR:host                | None                                           |
| OS-EXT-SRV-ATTR:hypervisor_hostname | None                                           |
| OS-EXT-SRV-ATTR:instance_name       |                                                |
| OS-EXT-STS:power_state              | NOSTATE                                        |
| OS-EXT-STS:task_state               | scheduling                                     |
| OS-EXT-STS:vm_state                 | building                                       |
| OS-SRV-USG:launched_at              | None                                           |
| OS-SRV-USG:terminated_at            | None                                           |
| accessIPv4                          |                                                |
| accessIPv6                          |                                                |
| addresses                           |                                                |
| adminPass                           | UG7izbYVjgNo                                   |
| config_drive                        |                                                |
| created                             | 2018-05-30T18:13:38Z                           |
| flavor                              | m1.tiny (7030eb41-1cd1-499d-b23f-c1e5bce6db3b) |
| hostId                              |                                                |
| id                                  | f9dceb00-2a5f-4430-a3f8-c4d187243597           |
| image                               | cirros2 (480d6322-6089-4e2f-aaa2-5810f5e3b0db) |
| key_name                            | None                                           |
| name                                | test_vm2                                       |
| progress                            | 0                                              |
| project_id                          | 948b76c981d248beb070aa399ec4e40f               |
| properties                          |                                                |
| security_groups                     | name='default'                                 |
| status                              | BUILD                                          |
| updated                             | 2018-05-30T18:13:38Z                           |
| user_id                             | fc353d7393644069a0abaad10e508a00               |
| volumes_attached                    |                                                |
+-------------------------------------+------------------------------------------------+

[root@localhost contrail-kolla-ansible]# ip route
default via 192.168.120.1 dev eth1
192.168.120.0/24 dev eth0 proto kernel scope link src 192.168.120.226

[root@localhost contrail-kolla-ansible]# ssh cirros@169.254.0.3
The authenticity of host '169.254.0.3 (169.254.0.3)' can't be established.
ECDSA key fingerprint is SHA256:CysQU7lFgg8m3WDwebbvKrQwyJ85VzLcoAAel9ItQwA.
ECDSA key fingerprint is MD5:3e:cf:d2:b8:7f:f0:1c:8d:b0:c6:82:be:0f:4f:e9:3f.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '169.254.0.3' (ECDSA) to the list of known hosts.
cirros@169.254.0.3's password:
$
$ ip -o a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1\    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
1: lo    inet 127.0.0.1/8 scope host lo\       valid_lft forever preferred_lft forever
1: lo    inet6 ::1/128 scope host \       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast qlen 1000\    link/ether 02:b7:60:c0:c5:bf brd ff:ff:ff:ff:ff:ff
2: eth0    inet 192.168.100.3/24 brd 192.168.100.255 scope global eth0\       valid_lft forever preferred_lft forever
2: eth0    inet6 fe80::b7:60ff:fec0:c5bf/64 scope link \       valid_lft forever preferred_lft forever

$ ping 192.168.100.4
PING 192.168.100.4 (192.168.100.4): 56 data bytes
64 bytes from 192.168.100.4: seq=0 ttl=64 time=2.695 ms
64 bytes from 192.168.100.4: seq=1 ttl=64 time=2.316 ms
^C
--- 192.168.100.4 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 2.316/2.505/2.695 ms
$ Connection to 169.254.0.3 closed.

```

