# Contrail 5.0.1 + OpenStack kolla ALL In One Install

## 0. Rqeuirement
Node: 2Node
  - K1: Deploy node
  - k2: Contrail and OpenStack node

Vagrant Box: centos/7

CentOS: CentOS Linux release 7.5.1804 (Core)

Kernel: Linux k2.lab 3.10.0-862.14.4.el7.x86_64 #1 SMP Wed Sep 26 15:12:11 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux

Network Interface: 3NIC

KVM Host: Enable Nested

contrail-ansible-deployer: branch master/commit 1314bf4ad0cdc33b31f48d814e153ecc09f0c20e

contrail-kolla-ansible: commit b0f7cc430494bb755ec0dc6711c9ba96ca449157

CONTAINER_REGISTRY: tungstenfabric

CONTRAIL_VERSION: r5.0.1

## 1. Create VM using by Vagrant
Please refer [How to install vagrant-libvirt](https://github.com/konono/tungsten-fabric-procedures/blob/master/vagrant/01_install_vagrant-libvirt.md)

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
  config.vm.define :k1 do |k1|
    k1.vm.box = "centos/7"
    k1.vm.box_version = "1809.01"
    k1.vm.box = "centos/7"
    k1.vm.network :public_network,
      ip: "172.27.116.132",
      netmask: "255.255.254.0",
      dev: "internet",
      mode: "bridge",
      type: "bridge"

    k1.vm.network :private_network,
      ip: '10.1.0.132',
      libvirt__network_name: "vnet01",
      libvirt__dhcp_enabled: false,
      libvirt__host_ip: '10.1.0.1',
      libvirt__forward_mode: 'none'
    k1.vm.hostname = "k1.lab"
    k1.vm.provision 'shell', inline: "sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config"
    k1.vm.provision 'shell', inline: "systemctl restart sshd"
    k1.vm.provision 'shell', inline: "yum install -y vim ntp epel-release git ansible net-tools python-devel gcc"
    k1.vm.provision 'shell', inline: "systemctl restart ntpd"
    k1.vm.provision 'shell', inline: "systemctl enable ntpd"
    k1.vm.provision 'shell', inline: "ip route del default && ip route add default via 172.27.116.1"
    k1.vm.provision 'shell', inline: "echo lab | passwd --stdin root"

  end
  config.vm.define :k2 do |k2|
    k2.vm.box = "centos/7"
    k2.vm.box_version = "1809.01"
    k2.vm.network :public_network,
      ip: "172.27.116.133",
      netmask: "255.255.254.0",
      dev: "internet",
      mode: "bridge",
      type: "bridge"

    k2.vm.network :private_network,
      ip: '10.1.0.133',
      libvirt__network_name: "vnet01",
      libvirt__dhcp_enabled: false,
      libvirt__host_ip: '10.1.0.1',
      libvirt__forward_mode: 'none'
    k2.vm.hostname = "k2.lab"
    k2.vm.provision 'shell', inline: "sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config"
    k2.vm.provision 'shell', inline: "systemctl restart sshd"
    k2.vm.provision 'shell', inline: "yum install -y vim ntp epel-release git ansible net-tools python-devel gcc"
    k2.vm.provision 'shell', inline: "systemctl restart ntpd"
    k2.vm.provision 'shell', inline: "systemctl enable ntpd"
    k2.vm.provision 'shell', inline: "ip route del default && ip route add default via 172.27.116.1"
    k2.vm.provision 'shell', inline: "echo lab | passwd --stdin root"

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
$ vagrant ssh k2
k2$ sudo -i
```

## 2. deploy

**You should following steps, If you do not use vagrant.**

### Prepare deploy
Target: **k1/k2**
```
k1/k2$ sudo su -

# Set password
k1/k2$ sudo sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config
k1/k2$ sudo systemctl restart sshd
k1/k2$ sudo passwd root

# Install require packages
k1/k2$ sudo yum install -y vim ntp epel-release git ansible net-tools python-devel

# Start NTP server
k1/k2$ sudo systemctl restart ntpd
k1/k2$ sudo systemctl enable ntpd
```


### 2.1. Configure /etc/hosts
Target: **k2**

**NW information in my environment**
```
======================================================================
[root@k2 ~]# ip -o a
1: lo    inet 127.0.0.1/8 scope host lo\       valid_lft forever preferred_lft forever
1: lo    inet6 ::1/128 scope host \       valid_lft forever preferred_lft forever
2: eth0    inet 192.168.120.126/24 brd 192.168.120.255 scope global dynamic eth0\       valid_lft 2765sec preferred_lft 2765sec
2: eth0    inet6 fe80::5054:ff:fe8b:9c69/64 scope link \       valid_lft forever preferred_lft forever
3: eth1    inet 172.27.116.133/23 brd 172.27.117.255 scope global eth1\       valid_lft forever preferred_lft forever
3: eth1    inet6 fe80::5054:ff:fe32:d3e9/64 scope link \       valid_lft forever preferred_lft forever
4: eth2    inet 10.1.0.133/24 brd 10.1.0.255 scope global eth2\       valid_lft forever preferred_lft forever
4: eth2    inet6 fe80::5054:ff:fed7:212b/64 scope link \       valid_lft forever preferred_lft forever
======================================================================
```

```
k2$ vim /etc/hosts

10.1.0.133      k2.lab k2
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4 k2.lab k2
172.27.116.133  k2.lab k2
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
k2$ exit
```

### 2.2. Deploy VM & Login
```
$ vagrant up
$ vagrant ssh k1
```

### 2.3 Clone repository 
Target: **k1**

```
k1$ cd ~/
k1$ git clone http://github.com/Juniper/contrail-ansible-deployer
k1$ cd contrail-ansible-deployer/
k1$ git checkout 1314bf4ad0cdc33b31f48d814e153ecc09f0c20e
```

### 2.4. Configuration instances.yaml

**NW information in my environment (before deploy)**
```
======================================================================
[root@k2 ~]# ip -o a
1: lo    inet 127.0.0.1/8 scope host lo\       valid_lft forever preferred_lft forever
1: lo    inet6 ::1/128 scope host \       valid_lft forever preferred_lft forever
2: eth0    inet 192.168.120.126/24 brd 192.168.120.255 scope global dynamic eth0\       valid_lft 2765sec preferred_lft 2765sec
2: eth0    inet6 fe80::5054:ff:fe8b:9c69/64 scope link \       valid_lft forever preferred_lft forever
3: eth1    inet 172.27.116.133/23 brd 172.27.117.255 scope global eth1\       valid_lft forever preferred_lft forever
3: eth1    inet6 fe80::5054:ff:fe32:d3e9/64 scope link \       valid_lft forever preferred_lft forever
4: eth2    inet 10.1.0.133/24 brd 10.1.0.255 scope global eth2\       valid_lft forever preferred_lft forever
4: eth2    inet6 fe80::5054:ff:fed7:212b/64 scope link \       valid_lft forever preferred_lft forever
======================================================================
```

**NW information in my environment (after deploy)**
```
======================================================================
[root@k2 ~]# ip -o a
1: lo    inet 127.0.0.1/8 scope host lo\       valid_lft forever preferred_lft forever
1: lo    inet6 ::1/128 scope host \       valid_lft forever preferred_lft forever
2: eth0    inet 192.168.120.126/24 brd 192.168.120.255 scope global dynamic eth0\       valid_lft 2539sec preferred_lft 2539sec
2: eth0    inet6 fe80::5054:ff:fe8b:9c69/64 scope link \       valid_lft forever preferred_lft forever
3: eth1    inet 172.27.116.20/32 scope global eth1\       valid_lft forever preferred_lft forever
3: eth1    inet6 fe80::5054:ff:fe32:d3e9/64 scope link \       valid_lft forever preferred_lft forever
4: eth2    inet 10.1.0.133/24 brd 10.1.0.255 scope global eth2\       valid_lft forever preferred_lft forever
4: eth2    inet6 fe80::5054:ff:fed7:212b/64 scope link \       valid_lft forever preferred_lft forever
5: docker0    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0\       valid_lft forever preferred_lft forever
9: vhost0    inet 172.27.116.133/23 brd 172.27.117.255 scope global vhost0\       valid_lft forever preferred_lft forever
9: vhost0    inet6 fe80::5054:ff:fe32:d3e9/64 scope link \       valid_lft forever preferred_lft forever
13: pkt0    inet6 fe80::38fb:4dff:febe:ecd/64 scope link \       valid_lft forever preferred_lft forever
======================================================================
```

```
k1$ vim contrail-ansible-deployer/config/instances.yaml
#######################################################################
#Eth0 - Internal API 192.168.120.0/24  - 192.168.120.126
#Eth1 - External API 172.27.116.0/23   - 172.27.116.133
#Eth2 - Tenant NW(vhost0) 10.1.0.0/24  - 10.1.0.133
#######################################################################

provider_config:
  bms:
    ssh_pwd: lab
    ssh_user: root
    ntpserver: 210.173.160.27
    domainsuffix: lab
#    ssh_public_key: /home/centos/.ssh/id_rsa.pub  # Optional. Not needed if ssh password is used.
#    ssh_private_key: /home/centos/.ssh/id_rsa     # Optional. Not needed if ssh password is used.
instances:
  bms1:
    provider: bms
    ip: 172.27.116.133
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
        VROUTER_GATEWAY: 172.27.116.1
        PHYSICAL_INTERFACE: eth1
      openstack_compute:
kolla_config:
  customize:
    nova.conf: |
      [libvirt]
      virt_type=kvm
      cpu_mode=host-passthrough
  kolla_globals:
    openstack_release: queens
    kolla_enable_tls_external: yes
    kolla_external_fqdn_cert: /etc/kolla/certificates/haproxy.pem
    keepalived_virtual_router_id: 21
    enable_haproxy: yes
    enable_swift: no
    enable_barbican: no
    enable_heat: yes
    enable_ironic: no
    enable_cinder: no
    enable_cinder_backend_nfs: no
    kolla_internal_vip_interface: eth1
    kolla_internal_vip_address: 172.27.116.20
    # kolla_external_vip_interface: eth1
    # kolla_external_vip_address: 172.27.116.20
    # heat/templates/heat.conf.j2:api_server = {{ contrail_api_interface_address }}
    contrail_api_interface_address: 10.1.0.133
    # neutron/templates/ContrailPlugin.ini.j2:api_server_ip = {{ opencontrail_api_server_ip }}
    opencontrail_api_server_ip: 10.1.0.133
  kolla_passwords:
    keystone_admin_password: lab
global_configuration:
  CONTAINER_REGISTRY: tungstenfabric
contrail_configuration:
  CONTRAIL_VERSION: r5.0.1
  UPGRADE_KERNEL: true
  CLOUD_ORCHESTRATOR: openstack
  CONTROLLER_NODES: 10.1.0.133
  CONTROL_NODES: 10.1.0.133
  ANALYTICSDB_NODES: 10.1.0.133
  WEBUI_NODES: 10.1.0.133
  ANALYTICS_NODES: 10.1.0.133
  ANALYTICS_API_LISTEN_IP : 10.1.0.133
  CONFIGDB_NODES: 10.1.0.133
  CONFIG_NODES: 10.1.0.133
  RABBITMQ_NODE_PORT: 5673
  AUTH_MODE: keystone
  KEYSTONE_AUTH_URL_VERSION: /v3
  KEYSTONE_AUTH_HOST: 172.27.116.20
  KEYSTONE_AUTH_PROTO: http
  CONFIG_NODEMGR__DEFAULTS__minimum_diskGB: 2
  DATABASE_NODEMGR__DEFAULTS__minimum_diskGB: 2
  CONFIG_DATABASE_NODEMGR__DEFAULTS__minimum_diskGB: 2

```

### 2.5. Configure instance
```
k1$ cd contrail-ansible-deployer
k1$ ansible-playbook -i inventory/ playbooks/configure_instances.yml
```

### 2.6. Deploy OpenStack
```
k1$ ansible-playbook -i inventory/ -e orchestrator=openstack playbooks/install_openstack.yml
```

### 2.7. Deploy Contrail
```
k1$ ansible-playbook -i inventory/ -e orchestrator=openstack playbooks/install_contrail.yml
```

### 2.8. Check contrail-status

```
[root@k1 ~]# contrail-status
Pod              Service         Original Name                          State    Status
                 redis           contrail-external-redis                running  Up 45 minutes
analytics        alarm-gen       contrail-analytics-alarm-gen           running  Up 41 minutes
analytics        api             contrail-analytics-api                 running  Up 41 minutes
analytics        collector       contrail-analytics-collector           running  Up 41 minutes
analytics        nodemgr         contrail-nodemgr                       running  Up 41 minutes
analytics        query-engine    contrail-analytics-query-engine        running  Up 41 minutes
analytics        snmp-collector  contrail-analytics-snmp-collector      running  Up 41 minutes
analytics        topology        contrail-analytics-topology            running  Up 41 minutes
config           api             contrail-controller-config-api         running  Up 43 minutes
config           device-manager  contrail-controller-config-devicemgr   running  Up 43 minutes
config           nodemgr         contrail-nodemgr                       running  Up 43 minutes
config           schema          contrail-controller-config-schema      running  Up 43 minutes
config           svc-monitor     contrail-controller-config-svcmonitor  running  Up 43 minutes
config-database  cassandra       contrail-external-cassandra            running  Up 44 minutes
config-database  nodemgr         contrail-nodemgr                       running  Up 44 minutes
config-database  rabbitmq        contrail-external-rabbitmq             running  Up 44 minutes
config-database  zookeeper       contrail-external-zookeeper            running  Up 44 minutes
control          control         contrail-controller-control-control    running  Up 42 minutes
control          dns             contrail-controller-control-dns        running  Up 42 minutes
control          named           contrail-controller-control-named      running  Up 42 minutes
control          nodemgr         contrail-nodemgr                       running  Up 42 minutes
database         cassandra       contrail-external-cassandra            running  Up 42 minutes
database         kafka           contrail-external-kafka                running  Up 42 minutes
database         nodemgr         contrail-nodemgr                       running  Up 42 minutes
database         zookeeper       contrail-external-zookeeper            running  Up 42 minutes
vrouter          agent           contrail-vrouter-agent                 running  Up 40 minutes
vrouter          nodemgr         contrail-nodemgr                       running  Up 40 minutes
webui            job             contrail-controller-webui-job          running  Up 43 minutes
webui            web             contrail-controller-webui-web          running  Up 43 minutes

WARNING: container with original name 'contrail-external-redis' have Pod or Service empty. Pod: '' / Service: 'redis'. Please pass NODE_TYPE with pod name to container's env

vrouter kernel module is PRESENT
== Contrail control ==
control: active
nodemgr: active
named: active
dns: active

== Contrail config-database ==
nodemgr: active
zookeeper: active
rabbitmq: active
cassandra: active

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
svc-monitor: active
nodemgr: active
device-manager: active
api: active
schema: active


```

:)

# Tips 

### How to start openstack

```
[root@localhost contrail-kolla-ansible]# pip install python-openstackclient
Collecting python-openstackclient

====== snip ====== 

Successfully installed PrettyTable-0.7.2 appdirs-1.4.3 asn1crypto-0.24.0 cffi-1.11.5 cliff-2.12.0 cmd2-0.8.7 contextlib2-0.5.5 cryptography-2.2.2 deprecation-2.0.2 dogpile.cache-0.6.5 futures-3.2.0 jsonpatch-1.23 jsonpointer-2.0 keystoneauth1-3.7.0 msgpack-0.5.6 munch-2.3.2 openstacksdk-0.13.0 os-client-config-1.31.1 os-service-types-1.2.0 osc-lib-1.10.0 oslo.serialization-2.25.0 packaging-17.1 pyOpenSSL-18.0.0 pyperclip-1.6.1 python-cinderclient-3.5.0 python-glanceclient-2.11.0 python-keystoneclient-3.16.0 python-novaclient-10.2.0 python-openstackclient-3.15.0 requestsexceptions-1.4.0 subprocess32-3.5.1 unicodecsv-0.14.1 warlock-1.3.0 wcwidth-0.1.7

[root@localhost contrail-kolla-ansible]# source /etc/kolla/admin-openrc.sh
[root@localhost contrail-kolla-ansible]#
[root@localhost contrail-kolla-ansible]# sudo yum -y install wget

====== snip ====== 

Installed:
  wget.x86_64 0:1.14-15.el7_4.1

Complete!

[root@localhost contrail-kolla-ansible]# wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
--2018-05-30 18:12:45--  http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
Resolving download.cirros-cloud.net (download.cirros-cloud.net)... 64.90.42.85, 2607:f298:6:a036::bd6:a72a
Connecting to download.cirros-cloud.net (download.cirros-cloud.net)|64.90.42.85|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 12716032 (12M) [text/plain]
Saving to: ‚Äòcirros-0.4.0-x86_64-disk.img

100%[=====================================================================================================================================================================================================================>] 12,716,032  2.65MB/s   in 12s

2018-05-30 18:12:57 (1.02 MB/s) - cirros-0.4.0-x86_64-disk.img saved [12716032/12716032]

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
cirros@169.254.0.3's password: [gocubsgo]
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
virsh snapshot-revert c01_k1 Init
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

## [OPTION] How to increase disk size for Vagrant box

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
