# Contrail 5.0 + OpenStack kolla ALL In One Install

## 0. Rqeuirement

Vagrant Box: centos/7

CentOS: CentOS Linux release 7.4.1708 (Core)

Kernel: 3.10.0-862.3.2.el7.x86_64 #1 SMP Mon May 21 23:36:36 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux

Network Interface: 3NIC

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
    k1.vm.network :public_network,
      ip: "172.27.116.93",
      netmask: "255.255.254.0",
      dev: "internet",
      mode: "bridge",
      type: "bridge"

    k1.vm.network :private_network,
      ip: '10.1.0.93',
      libvirt__network_name: "cm",
      libvirt__dhcp_enabled: false,
      libvirt__host_ip: '10.1.0.1',
      libvirt__forward_mode: 'none'

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
2: eth0    inet 192.168.120.202/24 brd 192.168.120.255 scope global eth0\       valid_lft forever preferred_lft forever
2: eth0    inet6 fe80::5054:ff:fef1:5155/64 scope link \       valid_lft forever preferred_lft forever
3: eth1    inet 172.27.116.93/23 brd 172.27.117.255 scope global eth1\       valid_lft forever preferred_lft forever
3: eth1    inet6 fe80::5054:ff:fe36:2650/64 scope link \       valid_lft forever preferred_lft forever
4: eth2    inet 10.1.0.93/24 brd 10.1.0.255 scope global eth2\       valid_lft forever preferred_lft forever
4: eth2    inet6 fe80::5054:ff:fedb:bf29/64 scope link \       valid_lft forever preferred_lft forever
======================================================================
```

```
$ vim /etc/hosts

127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
172.27.116.93  localhost.localdomain localhost
192.168.120.202  localhost.localdomain localhost
10.1.0.93  localhost.localdomain localhost
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
2: eth0    inet 192.168.120.202/24 brd 192.168.120.255 scope global eth0\       valid_lft forever preferred_lft forever
2: eth0    inet6 fe80::5054:ff:fef1:5155/64 scope link \       valid_lft forever preferred_lft forever
3: eth1    inet 172.27.116.93/23 brd 172.27.117.255 scope global eth1\       valid_lft forever preferred_lft forever
3: eth1    inet6 fe80::5054:ff:fe36:2650/64 scope link \       valid_lft forever preferred_lft forever
4: eth2    inet 10.1.0.93/24 brd 10.1.0.255 scope global eth2\       valid_lft forever preferred_lft forever
4: eth2    inet6 fe80::5054:ff:fedb:bf29/64 scope link \       valid_lft forever preferred_lft forever
======================================================================
```

**NW information in my environment (after deploy)**
```
======================================================================
[root@localhost ~]# ip -o a
1: lo    inet 127.0.0.1/8 scope host lo\       valid_lft forever preferred_lft forever
1: lo    inet6 ::1/128 scope host \       valid_lft forever preferred_lft forever
2: eth0    inet 192.168.120.202/24 brd 192.168.120.255 scope global dynamic eth0\       valid_lft 2232sec preferred_lft 2232sec
2: eth0    inet 192.168.120.120/32 scope global eth0\       valid_lft forever preferred_lft forever
2: eth0    inet6 fe80::5054:ff:fef1:5155/64 scope link \       valid_lft forever preferred_lft forever
3: eth1    inet 172.27.116.93/23 brd 172.27.117.255 scope global eth1\       valid_lft forever preferred_lft forever
3: eth1    inet 172.27.116.120/32 scope global eth1\       valid_lft forever preferred_lft forever
3: eth1    inet6 fe80::5054:ff:fe36:2650/64 scope link \       valid_lft forever preferred_lft forever
8: vhost0    inet 10.1.0.93/24 brd 10.1.0.255 scope global vhost0\       valid_lft forever preferred_lft forever
8: vhost0    inet6 fe80::5054:ff:fedb:bf29/64 scope link \       valid_lft forever preferred_lft forever
9: docker0    inet 172.17.0.1/16 scope global docker0\       valid_lft forever preferred_lft forever
======================================================================
```

```
$ vim contrail-ansible-deployer/config/instances.yaml
#######################################################################
#Eth0 - Internal API 192.168.120.0/24  - 192.168.120.202
#Eth1 - External API 172.27.116.0/23   - 172.27.116.93
#Eth2 - Tenant NW(vhost0) 10.1.0.0/24  - 10.1.0.93
#######################################################################

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
    ip: 172.27.116.93
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
        VROUTER_GATEWAY: 172.27.116.1
        PHYSICAL_INTERFACE: eth2
      openstack_compute:
kolla_config:
  customize:
    nova.conf: |
      [libvirt]
      virt_type=kvm
      cpu_mode=host-passthrough
  kolla_globals:
    openstack_release: ocata
    enable_haproxy: yes
    enable_swift: no
    enable_barbican: no
    enable_heat: yes
    enable_ironic: no
    kolla_internal_vip_interface: eth0
    kolla_internal_vip_address: 192.168.120.120
    kolla_external_vip_interface: eth1
    kolla_external_vip_address: 172.27.116.120
    # heat/templates/heat.conf.j2:api_server = {{ contrail_api_interface_address }}
    contrail_api_interface_address: 192.168.120.202
    # neutron/templates/ContrailPlugin.ini.j2:api_server_ip = {{ opencontrail_api_server_ip }}
    opencontrail_api_server_ip: 192.168.120.202
    keepalived_virtual_router_id: 33
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
  CONTROLLER_NODES: 192.168.120.202
  CONTROL_NODES: 192.168.120.202
  ANALYTICSDB_NODES: 192.168.120.202
  WEBUI_NODES: 192.168.120.202
  ANALYTICS_NODES: 192.168.120.202
  CONFIGDB_NODES: 192.168.120.202
  CONFIG_NODES: 192.168.120.202
  RABBITMQ_NODE_PORT: 5673
  AUTH_MODE: keystone
  KEYSTONE_AUTH_URL_VERSION: /v3

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
Collecting python-openstackclient
  Downloading https://files.pythonhosted.org/packages/33/84/c739b13aacc47d887cae58acc7dd921240918aa20a9add6ab64d932d1a6a/python_openstackclient-3.15.0-py2.py3-none-any.whl (823kB)
    100% |‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà| 829kB 1.1MB/s
Collecting keystoneauth1>=3.4.0 (from python-openstackclient)
  Downloading https://files.pythonhosted.org/packages/8a/c6/b305566d8a7060aa441ac46df95c07f12341428cff1b8fa93bdd5463426b/keystoneauth1-3.7.0-py2.py3-none-any.whl (288kB)
    100% |‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà| 296kB 3.6MB/s
Requirement already satisfied (use --upgrade to upgrade): oslo.i18n>=3.15.3 in /usr/lib/python2.7/site-packages (from python-openstackclient)
Requirement already satisfied (use --upgrade to upgrade): oslo.utils>=3.33.0 in /usr/lib/python2.7/site-packages (from python-openstackclient)
Collecting python-cinderclient>=3.3.0 (from python-openstackclient)
  Downloading https://files.pythonhosted.org/packages/4a/d8/bb1af0485b4e5b3ebe1253b1396e63861c23c7b5c9b0e0aa0807d105de55/python_cinderclient-3.5.0-py2.py3-none-any.whl (326kB)
    100% |‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà| 327kB 2.6MB/s
Requirement already satisfied (use --upgrade to upgrade): Babel!=2.4.0,>=2.3.4 in /usr/lib/python2.7/site-packages (from python-openstackclient)
Collecting python-novaclient>=9.1.0 (from python-openstackclient)
  Downloading https://files.pythonhosted.org/packages/95/d3/8bd00df0804a8253215312e89d5757de812b69f3a302e02c8766c2523ab1/python_novaclient-10.2.0-py2.py3-none-any.whl (308kB)
    100% |‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà| 317kB 3.6MB/s
Collecting python-keystoneclient>=3.8.0 (from python-openstackclient)
  Downloading https://files.pythonhosted.org/packages/f5/26/c21d9632e6892dfc23b5315e0221dba8b05b62d15d7488489a4b3953e854/python_keystoneclient-3.16.0-py2.py3-none-any.whl (376kB)
    100% |‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà| 378kB 2.2MB/s
Collecting cliff!=2.9.0,>=2.8.0 (from python-openstackclient)
  Downloading https://files.pythonhosted.org/packages/1a/80/8f60548a9685acfd027e74ccf1062984a6fe0cce7fc768ab315ff03e8139/cliff-2.12.0-py2-none-any.whl (71kB)
    100% |‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà| 81kB 6.9MB/s
Collecting osc-lib>=1.8.0 (from python-openstackclient)
  Downloading https://files.pythonhosted.org/packages/de/b7/91ed1a58756390fa006a5777ceb44578f021a9b67ffec0729f5037fff51a/osc_lib-1.10.0-py2-none-any.whl (81kB)
    100% |‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà| 81kB 7.9MB/s
Collecting openstacksdk>=0.11.2 (from python-openstackclient)
  Downloading https://files.pythonhosted.org/packages/fe/d5/d7292ccd2e6c4163f061c8648f805983387587a3e92e1f95a89204a28c06/openstacksdk-0.13.0-py2.py3-none-any.whl (997kB)
    100% |‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà| 1.0MB 1.0MB/s
Collecting python-glanceclient>=2.8.0 (from python-openstackclient)
  Downloading https://files.pythonhosted.org/packages/f8/74/14f925a9d623bf90d6cf7e08248778b4f606b2647418e1f1ebfba9ec933b/python_glanceclient-2.11.0-py2.py3-none-any.whl (179kB)
    100% |‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà| 184kB 5.4MB/s
Requirement already satisfied (use --upgrade to upgrade): six>=1.10.0 in /usr/lib/python2.7/site-packages (from python-openstackclient)
Requirement already satisfied (use --upgrade to upgrade): pbr!=2.1.0,>=2.0.0 in /usr/lib/python2.7/site-packages (from python-openstackclient)
Requirement already satisfied (use --upgrade to upgrade): iso8601>=0.1.11 in /usr/lib/python2.7/site-packages (from keystoneauth1>=3.4.0->python-openstackclient)
Requirement already satisfied (use --upgrade to upgrade): requests>=2.14.2 in /usr/lib/python2.7/site-packages (from keystoneauth1>=3.4.0->python-openstackclient)
Collecting os-service-types>=1.2.0 (from keystoneauth1>=3.4.0->python-openstackclient)
  Downloading https://files.pythonhosted.org/packages/c7/ec/7ef45820d4fa2964f0fea5b264bbb1594b68e330513a161ddcf0efd963e4/os_service_types-1.2.0-py2-none-any.whl
Requirement already satisfied (use --upgrade to upgrade): stevedore>=1.20.0 in /usr/lib/python2.7/site-packages (from keystoneauth1>=3.4.0->python-openstackclient)
Requirement already satisfied (use --upgrade to upgrade): pytz>=2013.6 in /usr/lib/python2.7/site-packages (from oslo.utils>=3.33.0->python-openstackclient)
Requirement already satisfied (use --upgrade to upgrade): netifaces>=0.10.4 in /usr/lib64/python2.7/site-packages (from oslo.utils>=3.33.0->python-openstackclient)
Requirement already satisfied (use --upgrade to upgrade): pyparsing>=2.1.0 in /usr/lib/python2.7/site-packages (from oslo.utils>=3.33.0->python-openstackclient)
Requirement already satisfied (use --upgrade to upgrade): funcsigs>=1.0.0; python_version == "2.7" or python_version == "2.6" in /usr/lib/python2.7/site-packages (from oslo.utils>=3.33.0->python-openstackclient)
Requirement already satisfied (use --upgrade to upgrade): netaddr>=0.7.18 in /usr/lib/python2.7/site-packages (from oslo.utils>=3.33.0->python-openstackclient)
Requirement already satisfied (use --upgrade to upgrade): monotonic>=0.6 in /usr/lib/python2.7/site-packages (from oslo.utils>=3.33.0->python-openstackclient)
Requirement already satisfied (use --upgrade to upgrade): debtcollector>=1.2.0 in /usr/lib/python2.7/site-packages (from oslo.utils>=3.33.0->python-openstackclient)
Collecting PrettyTable<0.8,>=0.7.1 (from python-cinderclient>=3.3.0->python-openstackclient)
  Downloading https://files.pythonhosted.org/packages/ef/30/4b0746848746ed5941f052479e7c23d2b56d174b82f4fd34a25e389831f5/prettytable-0.7.2.tar.bz2
Requirement already satisfied (use --upgrade to upgrade): simplejson>=3.5.1 in /usr/lib64/python2.7/site-packages (from python-cinderclient>=3.3.0->python-openstackclient)
Collecting oslo.serialization!=2.19.1,>=2.18.0 (from python-novaclient>=9.1.0->python-openstackclient)
  Downloading https://files.pythonhosted.org/packages/ca/c8/eb2fcb6cc97ac0e652a304855db37a0e433c2a217f9445e58ca67084075b/oslo.serialization-2.25.0-py2.py3-none-any.whl
Requirement already satisfied (use --upgrade to upgrade): oslo.config>=5.2.0 in /usr/lib/python2.7/site-packages (from python-keystoneclient>=3.8.0->python-openstackclient)
Collecting cmd2!=0.8.3,<0.9.0; python_version < "3.0" (from cliff!=2.9.0,>=2.8.0->python-openstackclient)
  Downloading https://files.pythonhosted.org/packages/3e/a0/839e37a31e7dfb349cbacdbb514054a71054010bc91185a1738e45485815/cmd2-0.8.7-py2.py3-none-any.whl (51kB)
    100% |‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà| 51kB 8.1MB/s
Collecting unicodecsv>=0.8.0; python_version < "3.0" (from cliff!=2.9.0,>=2.8.0->python-openstackclient)
  Downloading https://files.pythonhosted.org/packages/6f/a4/691ab63b17505a26096608cc309960b5a6bdf39e4ba1a793d5f9b1a53270/unicodecsv-0.14.1.tar.gz
Requirement already satisfied (use --upgrade to upgrade): PyYAML>=3.12 in /usr/lib64/python2.7/site-packages (from cliff!=2.9.0,>=2.8.0->python-openstackclient)
Collecting os-client-config>=1.28.0 (from osc-lib>=1.8.0->python-openstackclient)
  Downloading https://files.pythonhosted.org/packages/b7/e9/51de66c351be82c72928074de1820f5ed8a673496c45d619f9bb21ec69e9/os_client_config-1.31.1-py2.py3-none-any.whl
Collecting deprecation>=1.0 (from openstacksdk>=0.11.2->python-openstackclient)
  Downloading https://files.pythonhosted.org/packages/d1/98/caa4171c9ad4bb046b18bef84cc6e5d0ae0427baeee920eef53da83fae09/deprecation-2.0.2.tar.gz
Collecting dogpile.cache>=0.6.2 (from openstacksdk>=0.11.2->python-openstackclient)
  Downloading https://files.pythonhosted.org/packages/65/24/7bd97e9d486c37ac03ef6ae3a590db1a8e02183e5d7ce9071bcca9d86c44/dogpile.cache-0.6.5.tar.gz (320kB)
    100% |‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà| 327kB 3.3MB/s
Requirement already satisfied (use --upgrade to upgrade): ipaddress>=1.0.17; python_version < "3.3" in /usr/lib/python2.7/site-packages (from openstacksdk>=0.11.2->python-openstackclient)
Collecting futures>=3.0.0; python_version == "2.7" or python_version == "2.6" (from openstacksdk>=0.11.2->python-openstackclient)
  Downloading https://files.pythonhosted.org/packages/2d/99/b2c4e9d5a30f6471e410a146232b4118e697fa3ffc06d6a65efde84debd0/futures-3.2.0-py2-none-any.whl
Collecting requestsexceptions>=1.2.0 (from openstacksdk>=0.11.2->python-openstackclient)
  Downloading https://files.pythonhosted.org/packages/01/8c/49ca60ea8c907260da4662582c434bec98716177674e88df3fd340acf06d/requestsexceptions-1.4.0-py2.py3-none-any.whl
Requirement already satisfied (use --upgrade to upgrade): decorator>=3.4.0 in /usr/lib/python2.7/site-packages (from openstacksdk>=0.11.2->python-openstackclient)
Collecting munch>=2.1.0 (from openstacksdk>=0.11.2->python-openstackclient)
  Downloading https://files.pythonhosted.org/packages/68/f4/260ec98ea840757a0da09e0ed8135333d59b8dfebe9752a365b04857660a/munch-2.3.2.tar.gz
Collecting appdirs>=1.3.0 (from openstacksdk>=0.11.2->python-openstackclient)
  Downloading https://files.pythonhosted.org/packages/56/eb/810e700ed1349edde4cbdc1b2a21e28cdf115f9faf263f6bbf8447c1abf3/appdirs-1.4.3-py2.py3-none-any.whl
Collecting jsonpatch!=1.20,>=1.16 (from openstacksdk>=0.11.2->python-openstackclient)
  Downloading https://files.pythonhosted.org/packages/a0/e6/d50d526ae2218b765ddbdb2dda14d65e19f501ce07410b375bc43ad20b7a/jsonpatch-1.23-py2.py3-none-any.whl
Requirement already satisfied (use --upgrade to upgrade): jmespath>=0.9.0 in /usr/lib/python2.7/site-packages (from openstacksdk>=0.11.2->python-openstackclient)
Collecting warlock<2,>=1.2.0 (from python-glanceclient>=2.8.0->python-openstackclient)
  Downloading https://files.pythonhosted.org/packages/2d/40/9f01a5e1574dab946598793351d59c86f58209d182d229aaa545abb98894/warlock-1.3.0.tar.gz
Collecting pyOpenSSL>=17.1.0 (from python-glanceclient>=2.8.0->python-openstackclient)
  Downloading https://files.pythonhosted.org/packages/96/af/9d29e6bd40823061aea2e0574ccb2fcf72bfd6130ce53d32773ec375458c/pyOpenSSL-18.0.0-py2.py3-none-any.whl (53kB)
    100% |‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà| 61kB 9.0MB/s
Requirement already satisfied (use --upgrade to upgrade): wrapt>=1.7.0 in /usr/lib64/python2.7/site-packages (from python-glanceclient>=2.8.0->python-openstackclient)
Requirement already satisfied (use --upgrade to upgrade): urllib3<1.23,>=1.21.1 in /usr/lib/python2.7/site-packages (from requests>=2.14.2->keystoneauth1>=3.4.0->python-openstackclient)
Requirement already satisfied (use --upgrade to upgrade): idna<2.7,>=2.5 in /usr/lib/python2.7/site-packages (from requests>=2.14.2->keystoneauth1>=3.4.0->python-openstackclient)
Requirement already satisfied (use --upgrade to upgrade): chardet<3.1.0,>=3.0.2 in /usr/lib/python2.7/site-packages (from requests>=2.14.2->keystoneauth1>=3.4.0->python-openstackclient)
Requirement already satisfied (use --upgrade to upgrade): certifi>=2017.4.17 in /usr/lib/python2.7/site-packages (from requests>=2.14.2->keystoneauth1>=3.4.0->python-openstackclient)
Collecting msgpack>=0.4.0 (from oslo.serialization!=2.19.1,>=2.18.0->python-novaclient>=9.1.0->python-openstackclient)
  Downloading https://files.pythonhosted.org/packages/ff/0d/536fd0b2808dfbc67de46168dc706b5c9f9a1f5a803d96b8bc562a6d96c2/msgpack-0.5.6-cp27-cp27mu-manylinux1_x86_64.whl (295kB)
    100% |‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà| 296kB 3.3MB/s
Requirement already satisfied (use --upgrade to upgrade): enum34>=1.0.4; python_version == "2.7" or python_version == "2.6" or python_version == "3.3" in /usr/lib/python2.7/site-packages (from oslo.config>=5.2.0->python-keystoneclient>=3.8.0->python-openstackclient)
Requirement already satisfied (use --upgrade to upgrade): rfc3986>=0.3.1 in /usr/lib/python2.7/site-packages (from oslo.config>=5.2.0->python-keystoneclient>=3.8.0->python-openstackclient)
Collecting contextlib2; python_version < "3.5" (from cmd2!=0.8.3,<0.9.0; python_version < "3.0"->cliff!=2.9.0,>=2.8.0->python-openstackclient)
  Downloading https://files.pythonhosted.org/packages/a2/71/8273a7eeed0aff6a854237ab5453bc9aa67deb49df4832801c21f0ff3782/contextlib2-0.5.5-py2.py3-none-any.whl
Collecting pyperclip (from cmd2!=0.8.3,<0.9.0; python_version < "3.0"->cliff!=2.9.0,>=2.8.0->python-openstackclient)
  Downloading https://files.pythonhosted.org/packages/5c/6c/067f6a0d8a988e04e8934c5677ca9cbef144ac9e12c49267b820d1b2a063/pyperclip-1.6.1.tar.gz
Collecting wcwidth; sys_platform != "win32" (from cmd2!=0.8.3,<0.9.0; python_version < "3.0"->cliff!=2.9.0,>=2.8.0->python-openstackclient)
  Downloading https://files.pythonhosted.org/packages/7e/9f/526a6947247599b084ee5232e4f9190a38f398d7300d866af3ab571a5bfe/wcwidth-0.1.7-py2.py3-none-any.whl
Collecting subprocess32; python_version < "3.0" (from cmd2!=0.8.3,<0.9.0; python_version < "3.0"->cliff!=2.9.0,>=2.8.0->python-openstackclient)
  Downloading https://files.pythonhosted.org/packages/de/fb/fd3e91507021e2aecdb081d1b920082628d6b8869ead845e3e87b3d2e2ca/subprocess32-3.5.1.tar.gz (95kB)
    100% |‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà| 102kB 9.0MB/s
Collecting packaging (from deprecation>=1.0->openstacksdk>=0.11.2->python-openstackclient)
  Downloading https://files.pythonhosted.org/packages/ad/c2/b500ea05d5f9f361a562f089fc91f77ed3b4783e13a08a3daf82069b1224/packaging-17.1-py2.py3-none-any.whl
Collecting jsonpointer>=1.9 (from jsonpatch!=1.20,>=1.16->openstacksdk>=0.11.2->python-openstackclient)
  Downloading https://files.pythonhosted.org/packages/18/b0/a80d29577c08eea401659254dfaed87f1af45272899e1812d7e01b679bc5/jsonpointer-2.0-py2.py3-none-any.whl
Requirement already satisfied (use --upgrade to upgrade): jsonschema<3,>=0.7 in /usr/lib/python2.7/site-packages (from warlock<2,>=1.2.0->python-glanceclient>=2.8.0->python-openstackclient)
Collecting cryptography>=2.2.1 (from pyOpenSSL>=17.1.0->python-glanceclient>=2.8.0->python-openstackclient)
  Downloading https://files.pythonhosted.org/packages/dd/c2/3a5bfefb25690725824ade71e6b65449f0a9f4b29702cce10560f786ebf6/cryptography-2.2.2-cp27-cp27mu-manylinux1_x86_64.whl (2.2MB)
    100% |‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà| 2.2MB 380kB/s
Requirement already satisfied (use --upgrade to upgrade): functools32; python_version == "2.7" in /usr/lib/python2.7/site-packages (from jsonschema<3,>=0.7->warlock<2,>=1.2.0->python-glanceclient>=2.8.0->python-openstackclient)
Collecting cffi>=1.7; platform_python_implementation != "PyPy" (from cryptography>=2.2.1->pyOpenSSL>=17.1.0->python-glanceclient>=2.8.0->python-openstackclient)
  Downloading https://files.pythonhosted.org/packages/14/dd/3e7a1e1280e7d767bd3fa15791759c91ec19058ebe31217fe66f3e9a8c49/cffi-1.11.5-cp27-cp27mu-manylinux1_x86_64.whl (407kB)
    100% |‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà| 409kB 1.9MB/s
Collecting asn1crypto>=0.21.0 (from cryptography>=2.2.1->pyOpenSSL>=17.1.0->python-glanceclient>=2.8.0->python-openstackclient)
  Downloading https://files.pythonhosted.org/packages/ea/cd/35485615f45f30a510576f1a56d1e0a7ad7bd8ab5ed7cdc600ef7cd06222/asn1crypto-0.24.0-py2.py3-none-any.whl (101kB)
    100% |‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà‚ñà| 102kB 4.0MB/s
Requirement already satisfied (use --upgrade to upgrade): pycparser in /usr/lib/python2.7/site-packages (from cffi>=1.7; platform_python_implementation != "PyPy"->cryptography>=2.2.1->pyOpenSSL>=17.1.0->python-glanceclient>=2.8.0->python-openstackclient)
Installing collected packages: os-service-types, keystoneauth1, PrettyTable, python-cinderclient, msgpack, oslo.serialization, python-novaclient, python-keystoneclient, contextlib2, pyperclip, wcwidth, subprocess32, cmd2, unicodecsv, cliff, packaging, deprecation, dogpile.cache, futures, requestsexceptions, munch, appdirs, jsonpointer, jsonpatch, openstacksdk, os-client-config, osc-lib, warlock, cffi, asn1crypto, cryptography, pyOpenSSL, python-glanceclient, python-openstackclient
  Running setup.py install for PrettyTable ... done
  Running setup.py install for pyperclip ... done
  Running setup.py install for subprocess32 ... done
  Running setup.py install for unicodecsv ... done
  Running setup.py install for deprecation ... done
  Running setup.py install for dogpile.cache ... done
  Running setup.py install for munch ... done
  Running setup.py install for warlock ... done
  Found existing installation: cffi 1.6.0
    Uninstalling cffi-1.6.0:
      Successfully uninstalled cffi-1.6.0
  Found existing installation: cryptography 1.7.2
    Uninstalling cryptography-1.7.2:
      Successfully uninstalled cryptography-1.7.2
Successfully installed PrettyTable-0.7.2 appdirs-1.4.3 asn1crypto-0.24.0 cffi-1.11.5 cliff-2.12.0 cmd2-0.8.7 contextlib2-0.5.5 cryptography-2.2.2 deprecation-2.0.2 dogpile.cache-0.6.5 futures-3.2.0 jsonpatch-1.23 jsonpointer-2.0 keystoneauth1-3.7.0 msgpack-0.5.6 munch-2.3.2 openstacksdk-0.13.0 os-client-config-1.31.1 os-service-types-1.2.0 osc-lib-1.10.0 oslo.serialization-2.25.0 packaging-17.1 pyOpenSSL-18.0.0 pyperclip-1.6.1 python-cinderclient-3.5.0 python-glanceclient-2.11.0 python-keystoneclient-3.16.0 python-novaclient-10.2.0 python-openstackclient-3.15.0 requestsexceptions-1.4.0 subprocess32-3.5.1 unicodecsv-0.14.1 warlock-1.3.0 wcwidth-0.1.7
You are using pip version 8.1.2, however version 10.0.1 is available.
You should consider upgrading via the 'pip install --upgrade pip' command.
[root@localhost contrail-kolla-ansible]# source /etc/kolla/admin-openrc.sh
[root@localhost contrail-kolla-ansible]#
[root@localhost contrail-kolla-ansible]# sudo yum -y install wget
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: www.ftp.ne.jp
 * epel: mirror.dmmlabs.jp
 * extras: www.ftp.ne.jp
 * updates: www.ftp.ne.jp
No package http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img available.
Resolving Dependencies
--> Running transaction check
---> Package wget.x86_64 0:1.14-15.el7_4.1 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

===============================================================================================================================================================================================================================================================
 Package                                                   Arch                                                        Version                                                                 Repository                                                 Size
===============================================================================================================================================================================================================================================================
Installing:
 wget                                                      x86_64                                                      1.14-15.el7_4.1                                                         base                                                      547 k

Transaction Summary
===============================================================================================================================================================================================================================================================
Install  1 Package

Total download size: 547 k
Installed size: 2.0 M
Downloading packages:
wget-1.14-15.el7_4.1.x86_64.rpm                                                                                                                                                                                                         | 547 kB  00:00:00
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
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
[root@localhost contrail-kolla-ansible]#
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
default via 172.27.116.1 dev eth1
10.1.0.0/24 dev vhost0 proto kernel scope link src 10.1.0.93
169.254.0.0/16 dev eth0 scope link metric 1002
169.254.0.0/16 dev eth1 scope link metric 1003
169.254.0.1 dev vhost0 proto 109 scope link
169.254.0.3 dev vhost0 proto 109 scope link
169.254.0.4 dev vhost0 proto 109 scope link
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1
172.27.116.0/23 dev eth1 proto kernel scope link src 172.27.116.93
192.168.120.0/24 dev eth0 proto kernel scope link src 192.168.120.202
[root@localhost contrail-kolla-ansible]# ssh cirros@169.254.0.3
The authenticity of host '169.254.0.3 (169.254.0.3)' can't be established.
ECDSA key fingerprint is SHA256:CysQU7lFgg8m3WDwebbvKrQwyJ85VzLcoAAel9ItQwA.
ECDSA key fingerprint is MD5:3e:cf:d2:b8:7f:f0:1c:8d:b0:c6:82:be:0f:4f:e9:3f.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '169.254.0.3' (ECDSA) to the list of known hosts.
cirros@169.254.0.3's password:
$
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

### Create Snapshot and revert tool
```
vagrant halt
virsh snapshot-create-as c01_k1 Init


cat <<EOF> revert.sh
virsh snapshot-revert c02_k1 Init
sleep 5
vagrant up
EOF
```

### Recreate Snapshot
```
virsh snapshot-delete c01_k1 Init
virsh snapshot-create-as c01_k1 Init
```

### How to use openstack command
```
e.g.
docker exec kolla_toolbox openstack --os-interface internal --os-auth-url http://xxx.xxx.xxx.xxx:35357/v3 --os-identity-api-version 3 --os-project-domain-name default --os-tenant-name admin --os-username admin --os-password lab --os-user-domain-name default compute service list -f json --service nova-compute
```

# Trouble-shooting
### 1. vRouter-agent is not stiil down

```
contrail-status

====== Snip ======
== Contrail vrouter ==
nodemgr: active
agent: inactive <--- Inactive Stat is wrong
====== Snip ======
```

#### 1.1. Check lsmod
```
[root@localhost ~]# lsmod |grep vrouter
vrouter               460243  2
```

#### 1.2. Check vhost interface
```
[root@localhost ~]# ip -o a
1: lo    inet 127.0.0.1/8 scope host lo\       valid_lft forever preferred_lft forever
1: lo    inet6 ::1/128 scope host \       valid_lft forever preferred_lft forever
2: eth0    inet6 fe80::5054:ff:fef1:5155/64 scope link \       valid_lft forever preferred_lft forever
3: eth1    inet 172.27.116.93/23 brd 172.27.117.255 scope global eth1\       valid_lft forever preferred_lft forever
3: eth1    inet 172.27.116.120/32 scope global eth1\       valid_lft forever preferred_lft forever
3: eth1    inet6 fe80::5054:ff:fe36:2650/64 scope link \       valid_lft forever preferred_lft forever
4: eth2    inet 10.1.0.93/24 brd 10.1.0.255 scope global eth2\       valid_lft forever preferred_lft forever
4: eth2    inet 10.1.0.120/32 scope global eth2\       valid_lft forever preferred_lft forever
4: eth2    inet6 fe80::5054:ff:fedb:bf29/64 scope link \       valid_lft forever preferred_lft forever
5: docker0    inet 172.17.0.1/16 scope global docker0\       valid_lft forever preferred_lft forever
9: vhost0    inet 192.168.120.202/24 brd 192.168.120.255 scope global dynamic vhost0\       valid_lft 3103sec preferred_lft 3103sec
9: vhost0    inet6 fe80::5054:ff:fef1:5155/64 scope link \       valid_lft forever preferred_lft forever
11: pkt0    inet6 fe80::2039:aaff:fee3:9047/64 scope link \       valid_lft forever preferred_lft forever
```

#### 1.3. Check vRouter agent container logs
```
[root@localhost ~]# docker logs vrouter_vrouter-agent_1
INFO: agent started in kernel mode
Device "vhost0" does not exist.
cat: /sys/class/net/vhost0/address: No such file or directory
INFO: creating vhost0 for nic mode: nic: eth0, mac=52:54:00:f1:51:55
              total        used        free      shared  buff/cache   available
Mem:            62G         24G         27G         12M         11G         37G
Swap:            0B          0B          0B
              total        used        free      shared  buff/cache   available
Mem:            62G         23G         36G         12M        2.5G         38G
Swap:            0B          0B          0B
INFO: load vrouter kernel module
insmod /lib/modules/3.10.0-693.21.1.el7.x86_64/kernel/net/vrouter/vrouter.ko
INFO: creating ifcfg-vhost0 and initialize it via ifup
/etc/sysconfig/network-scripts /
/

Determining IP information for vhost0... done.
INFO: Physical interface: eth0, mac=52:54:00:f1:51:55, pci=0000:00:05.0
INFO: vhost0 cidr 192.168.120.202/24, gateway 192.168.120.1
INFO: Preparing /etc/contrail/contrail-vrouter-agent.conf
INFO: /etc/contrail/contrail-vrouter-agent.conf
[CONTROL-NODE]
servers=172.27.116.93:5269


[DEFAULT]
collectors=172.27.116.93:8086
log_file=/var/log/contrail/contrail-vrouter-agent.log
log_level=SYS_NOTICE
log_local=1

xmpp_dns_auth_enable=False
xmpp_auth_enable=False


physical_interface_mac = 52:54:00:f1:51:55

tsn_servers = []

[SANDESH]
introspect_ssl_enable=False
sandesh_ssl_enable=False

[NETWORKS]
control_network_ip=172.27.116.93

[DNS]
servers=172.27.116.93:53

[METADATA]
metadata_proxy_secret=contrail


[VIRTUAL-HOST-INTERFACE]
name=vhost0
ip=192.168.120.202/24
physical_interface=eth0
gateway=192.168.120.1
compute_node_address=192.168.120.202

[SERVICE-INSTANCE]
netns_command=/usr/bin/opencontrail-vrouter-netns
docker_command=/usr/bin/opencontrail-vrouter-docker

[HYPERVISOR]
type = kvm


[FLOWS]
fabric_snat_hash_table_size = 4096





INFO: Preparing /etc/contrail/contrail-lbaas-auth.conf
Config file </etc/contrail/contrail-vrouter-agent.conf> parsing completed.
log4cplus:ERROR No appenders could be found for logger (SANDESH).
log4cplus:ERROR Please initialize the log4cplus system properly.
INFO: agent started in kernel mode
INFO: vhost0 is already up
INFO: Physical interface: eth0, mac=52:54:00:f1:51:55, pci=0000:00:05.0
INFO: vhost0 cidr 192.168.120.202/24, gateway 192.168.120.1
INFO: Preparing /etc/contrail/contrail-vrouter-agent.conf
INFO: /etc/contrail/contrail-vrouter-agent.conf
[CONTROL-NODE]
servers=172.27.116.93:5269


[DEFAULT]
collectors=172.27.116.93:8086
log_file=/var/log/contrail/contrail-vrouter-agent.log
log_level=SYS_NOTICE
log_local=1

xmpp_dns_auth_enable=False
xmpp_auth_enable=False


physical_interface_mac = 52:54:00:f1:51:55

tsn_servers = []

[SANDESH]
introspect_ssl_enable=False
sandesh_ssl_enable=False

[NETWORKS]
control_network_ip=172.27.116.93

[DNS]
servers=172.27.116.93:53

[METADATA]
metadata_proxy_secret=contrail


[VIRTUAL-HOST-INTERFACE]
name=vhost0
ip=192.168.120.202/24
physical_interface=eth0
gateway=192.168.120.1
compute_node_address=192.168.120.202

[SERVICE-INSTANCE]
netns_command=/usr/bin/opencontrail-vrouter-netns
docker_command=/usr/bin/opencontrail-vrouter-docker

[HYPERVISOR]
type = kvm


[FLOWS]
fabric_snat_hash_table_size = 4096





INFO: Preparing /etc/contrail/contrail-lbaas-auth.conf
Config file </etc/contrail/contrail-vrouter-agent.conf> parsing completed.
log4cplus:ERROR No appenders could be found for logger (SANDESH).
log4cplus:ERROR Please initialize the log4cplus system properly.
```

#### 1.4. Restart vRouter agent container
```
[root@localhost ~]# docker kill vrouter_vrouter-agent_1
[root@localhost ~]# docker start vrouter_vrouter-agent_1
```
