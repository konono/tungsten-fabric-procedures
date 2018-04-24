# Contrail 5.0 + OpenStack kolla ALL In One Install

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

### 2.5 Clone repository 
```
$ cd ~/
$ git clone http://github.com/Juniper/contrail-ansible-deployer
$ cd contrail-ansible-deployer/
```

### 2.6. Apply patch
```
sed -i 's/roles.item /roles[item] /g' playbooks/roles/create_openstack_config/tasks/host_params.yml
```

***[Patch information](https://github.com/Juniper/contrail-ansible-deployer/pull/14)***

### 2.7. Configuration instances.yaml

**NW information for my environment**
```
======================================================================
[root@localhost ~]# ip -o a
1: lo    inet 127.0.0.1/8 scope host lo\       valid_lft forever preferred_lft forever
1: lo    inet6 ::1/128 scope host \       valid_lft forever preferred_lft forever
2: eth0    inet 192.168.120.202/24 brd 192.168.120.255 scope global eth0\       valid_lft forever preferred_lft forever
2: eth0    inet6 fe80::5054:ff:fef1:5155/64 scope link \       valid_lft forever preferred_lft forever
3: eth1    inet 172.27.116.93/23 brd 172.27.117.255 scope global eth1\       valid_lft forever preferred_lft forever
3: eth1    inet 172.27.116.120/32 scope global eth1\       valid_lft forever preferred_lft forever
3: eth1    inet6 fe80::5054:ff:fe36:2650/64 scope link \       valid_lft forever preferred_lft forever
4: eth2    inet 10.1.0.93/24 brd 10.1.0.255 scope global eth2\       valid_lft forever preferred_lft forever
4: eth2    inet 10.1.0.120/32 scope global eth2\       valid_lft forever preferred_lft forever
4: eth2    inet6 fe80::5054:ff:fedb:bf29/64 scope link \       valid_lft forever preferred_lft forever
======================================================================
```

```
$ vim config/instances.yaml

#######################################################################
#Eth0 - Tenant NW(vhost0)
#Eth1 - Internal API
#Eth2 - External API
#######################################################################

provider_config:
  bms:
    ssh_pwd: lab
    ssh_user: root
    domainsuffix: local
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
          network_interface: eth1
        openstack_storage:
        openstack_monitoring:
        vrouter:
          VROUTER_GATEWAY: 192.168.120.1
          IPFABRIC_SERVICE_HOST: 192.168.120.202
        openstack_compute:
kolla_config:
  customize:
    nova.conf: |
      [libvirt]
      virt_type=kvm
     #virt_type=qemu
      cpu_mode=host-passthrough
     #cpu_mode=none
  kolla_globals:
    openstack_release: ocata
    contrail_api_interface_address: 172.27.116.120
    enable_haproxy: yes
    enable_swift: no
    enable_barbican: no
    enable_heat: no
    enable_ironic: no
    kolla_internal_vip_interface: eth1
    kolla_internal_vip_address: 172.27.116.120
    kolla_external_vip_interface: eth2
    kolla_external_vip_address: 10.1.0.120
  kolla_passwords:
    keystone_admin_password: lab
global_configuration:
  CONTAINER_REGISTRY: opencontrailnightly
contrail_configuration:
  CONTRAIL_VERSION: latest
  CLOUD_ORCHESTRATOR: openstack
  CONTROL_DATA_NET_LIST: 192.168.120.0/24
  CONTROLLER_NODES: 172.27.116.93
  CONTROL_NODES: 172.27.116.93
  ANALYTICSDB_NODES: 172.27.116.93
  WEBUI_NODES: 172.27.116.93
  ANALYTICS_NODES: 172.27.116.93
  CONFIGDB_NODES: 172.27.116.93
  CONFIG_NODES: 172.27.116.93
  RABBITMQ_NODE_PORT: 5673
  AUTH_MODE: keystone
  KEYSTONE_AUTH_URL_VERSION: /v3

```

### 2.8. Configure instance
```
$ cd contrail-ansible-deployer
$ ansible-playbook -i inventory/ playbooks/configure_instances.yml
```

### 2.9. Apply patch
```
$ sed -i 's/use_neutron = True//g' ~/contrail-kolla-ansible/ansible/roles/nova/templates/nova.conf.j2
```

***[Patch information](https://bugs.launchpad.net/kolla-ansible/+bug/1651665)***

### 2.10. Deploy Contrail+OpenStack
```
$ ansible-playbook -i inventory/ -e orchestrator=openstack playbooks/install_contrail.yml
```

### 2.11. Check contrail-status

```
[root@localhost ~]# contrail-status
Pod        Service         Original Name                          State    Status
analytics  alarm-gen       contrail-analytics-alarm-gen           running  Up 2 days
analytics  api             contrail-analytics-api                 running  Up 2 days
analytics  collector       contrail-analytics-collector           running  Up 2 days
analytics  nodemgr         contrail-nodemgr                       running  Up 2 days
analytics  query-engine    contrail-analytics-query-engine        running  Up 2 days
analytics  snmp-collector  contrail-analytics-snmp-collector      running  Up 2 days
analytics  topology        contrail-analytics-topology            running  Up 2 days
config     api             contrail-controller-config-api         running  Up 2 days
config     cassandra       contrail-external-cassandra            running  Up 2 days
config     device-manager  contrail-controller-config-devicemgr   running  Up 2 days
config     nodemgr         contrail-nodemgr                       running  Up 2 days
config     rabbitmq        contrail-external-rabbitmq             running  Up 2 days
config     schema          contrail-controller-config-schema      running  Up 2 days
config     svc-monitor     contrail-controller-config-svcmonitor  running  Up 2 days
config     zookeeper       contrail-external-zookeeper            running  Up 2 days
control    control         contrail-controller-control-control    running  Up 2 days
control    dns             contrail-controller-control-dns        running  Up 2 days
control    named           contrail-controller-control-named      running  Up 2 days
control    nodemgr         contrail-nodemgr                       running  Up 2 days
database   cassandra       contrail-external-cassandra            running  Up 2 days
database   kafka           contrail-external-kafka                running  Up 2 days
database   nodemgr         contrail-nodemgr                       running  Up About an hour
database   zookeeper       contrail-external-zookeeper            running  Up 2 days
vrouter    agent           contrail-vrouter-agent                 running  Up 2 days
vrouter    nodemgr         contrail-nodemgr                       running  Up 2 days
webui      job             contrail-controller-webui-job          running  Up 2 days
webui      web             contrail-controller-webui-web          running  Up 2 days

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
snmp-collector: active
query-engine: active
api: active
alarm-gen: active
nodemgr: active
collector: active
topology: active

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
