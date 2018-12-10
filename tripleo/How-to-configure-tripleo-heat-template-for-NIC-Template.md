# 今からでもできる！TripleO Heat Template入門 NIC Template編
## NIC Template Configuration

### **編集対象ファイル**
- environments/contrail/contrail-net.yaml
- network/config/contrail/controller-nic-config.yaml
- network/config/contrail/contrail-controller-nic-config.yaml
- network/config/contrail/contrail-controller-nic-config.yaml
- network/config/contrail/compute-nic-config.yaml
- network/config/contrail/contrail-dpdk-nic-config.yaml
- network/config/contrail/contrail-sriov-nic-config.yaml
- network/config/contrail/contrail-tsn-nic-config.yaml
- network/config/contrail/contrail-controller-nic-config.yaml

### contrail-net.yaml
```
resource_registry:
  OS::TripleO::Controller::Net::SoftwareConfig: ../../network/config/contrail/controller-nic-config.yaml
  OS::TripleO::ContrailController::Net::SoftwareConfig: ../../network/config/contrail/contrail-controller-nic-config.yaml
  OS::TripleO::ContrailControlOnly::Net::SoftwareConfig: ../../network/config/contrail/contrail-controller-nic-config.yaml
  OS::TripleO::Compute::Net::SoftwareConfig: ../../network/config/contrail/compute-nic-config.yaml
  OS::TripleO::ContrailDpdk::Net::SoftwareConfig: ../../network/config/contrail/contrail-dpdk-nic-config.yaml
  OS::TripleO::ContrailSriov::Net::SoftwareConfig: ../../network/config/contrail/contrail-sriov-nic-config.yaml
  OS::TripleO::ContrailTsn::Net::SoftwareConfig: ../../network/config/contrail/contrail-tsn-nic-config.yaml
  OS::TripleO::ContrailAnalytics::Net::SoftwareConfig: ../../network/config/contrail/contrail-controller-nic-config.yaml

### ここで設定されている値は、NIC Templateの中でget_param: [Key]で呼び出される値である。
parameter_defaults:
  # Customize all these values to match the local environment
  ### 各セグメントのSubenetをここで記述
  TenantNetCidr: 172.16.4.0/24
  InternalApiNetCidr: 172.27.116.0/24
  ExternalNetCidr: 10.0.0.0/23
  StorageNetCidr: 10.0.2.0/24
  StorageMgmtNetCidr: 10.0.3.0/24

  # CIDR subnet mask length for provisioning network
  ControlPlaneSubnetCidr: '24'
  # Allocation pools

  ### 各セグメントはDHCPでIPが払い出されるので、Poolの設定を行う
  TenantAllocationPools: [{'start': '172.16.4.55', 'end': '172.16.4.59'}]
  InternalApiAllocationPools: [{'start': '172.27.116.55', 'end': '172.27.116.59'}]
  ExternalAllocationPools: [{'start': '10.0.0.10', 'end': '10.0.0.200'}]
  StorageAllocationPools: [{'start': '10.0.2.10', 'end': '10.0.2.200'}]
  StorageMgmtAllocationPools: [{'start': '10.0.3.10', 'end': '10.0.3.200'}]

  # Routes
  ### 各セグメントのDefaultRouteの定義を行う、利用しないセグメントでも定義は必要だったりもする
  ControlPlaneDefaultRoute: 10.0.9.254
  InternalApiDefaultRoute: 172.27.116.1
  ExternalInterfaceDefaultRoute: 10.0.0.254

  # Vlans
  ### 各セグメントのVLANの定義を行う
  InternalApiNetworkVlanID: 362
  ExternalNetworkVlanID: 361
  StorageNetworkVlanID: 363
  StorageMgmtNetworkVlanID: 364
  TenantNetworkVlanID: 365
  # Services

  ### EC2MetadataIPはUndercloudのIPを指定、残りはお好みのDNSとNTPを入力
  EC2MetadataIp: 172.16.1.1  # Generally the IP of the Undercloud
  DnsServers: ["8.8.8.8"]
  NtpServer: ntp.jst.mfeed.ad.jp
```


### NIC Template Configurations

#### ControlPlane configuration

```
- type: interface
  name: eth0
  use_dhcp: false
  dns_servers:
    get_param: DnsServers
  addresses:
  - ip_netmask:
      list_join:
        - '/'
        - - get_param: ControlPlaneIp
          - get_param: ControlPlaneSubnetCidr
  routes:
  - ip_netmask: 169.254.169.254/32
    next_hop:
      get_param: EC2MetadataIp
```

#### Controller, Contrail Controller and Compute(without vrouter interface) NIC Configuration

In addition to the standard NIC configuration supports the following modes:
- Standard
- VLAN
- Bond
- Bond + VLAN
- bridge
- bridge + VLAN

#### Standard
```
- type: interface
  name: nic1
  addresses:
  - ip_netmask:
      get_param: TenantIpSubnet
```

#### VLAN
```
- type: vlan
  vlan_id:
    get_param: TenantNetworkVlanID
  device: nic1
  addresses:
  - ip_netmask:
      get_param: TenantIpSubnet
```

#### Bond
```
- type: linux_bond
  name: bond0
  bonding_options: "mode=4 xmit_hash_policy=layer2+3"
  use_dhcp: false
  members:
    -
      type: interface
      name: nic1
    -
      type: interface
      name: nic2
  addresses:
  - ip_netmask:
      get_param: TenantIpSubnet
```

#### Bond + VLAN
```
- type: linux_bond
  name: bond0
  bonding_options: "mode=4 xmit_hash_policy=layer2+3"
  use_dhcp: false
  members:
    -
      type: interface
      name: nic1
    -
      type: interface
      name: nic2

- type: vlan
  vlan_id:
    get_param: TenantNetworkVlanID
  device: bond0
  addresses:
  - ip_netmask:
      get_param: TenantIpSubnet
```

#### Bridge
```
- type: linux_bridge
  name: br0
  use_dhcp: false
  members:
    -
      type: interface
      name: nic1
```

#### Bridge + VLAN
```
- type: linux_bridge
  name: br0
  use_dhcp: false
  members:
    -
      type: interface
      name: nic1
- type: vlan
  vlan_id:
    get_param: TenantNetworkVlanID
  device: br0
  addresses:
  - ip_netmask:
      get_param: TenantIpSubnet
```

#### Advanced vRouter Kernel Mode Configurations
Network interface configuration

There are NIC configuration files per role.

In addition to the standard NIC configuration, the vRouter kernel mode supports the following modes:
- VLAN
- Bond
- Bond + VLAN

The snippets below only shows the relevant section of the NIC configuration for each mode.

#### VLAN
```
- type: vlan
  vlan_id:
    get_param: TenantNetworkVlanID
  device: nic2
- type: contrail_vrouter
  name: vhost0
  use_dhcp: false
  members:
    -
      type: interface
      name:
        str_replace:
          template: vlanVLANID
          params:
            VLANID: {get_param: TenantNetworkVlanID}
      use_dhcp: false
  addresses:
  - ip_netmask:
      get_param: TenantIpSubnet
```

#### Bond
```
- type: linux_bond
  name: bond0
  bonding_options: "mode=4 xmit_hash_policy=layer2+3"
  use_dhcp: false
  members:
    -
      type: interface
      name: nic2
    -
      type: interface
      name: nic3
- type: contrail_vrouter
  name: vhost0
  use_dhcp: false
  members:
    -
      type: interface
      name: bond0
      use_dhcp: false
  addresses:
  - ip_netmask:
      get_param: TenantIpSubnet
```

#### Bond + VLAN
```
- type: linux_bond
  name: bond0
  bonding_options: "mode=4 xmit_hash_policy=layer2+3"
  use_dhcp: false
  members:
    -
      type: interface
      name: nic2
    -
      type: interface
      name: nic3
- type: vlan
  vlan_id:
    get_param: TenantNetworkVlanID
  device: bond0
- type: contrail_vrouter
  name: vhost0
  use_dhcp: false
  members:
    -
      type: interface
      name:
        str_replace:
          template: vlanVLANID
          params:
            VLANID: {get_param: TenantNetworkVlanID}
      use_dhcp: false
  addresses:
  - ip_netmask:
      get_param: TenantIpSubnet
```

#### Advanced vRouter DPDK Mode Configurations

In addition to the standard NIC configuration, the vRouter DPDK mode supports the following modes:

- Standard
- VLAN
- Bond
- Bond + VLAN

Enable the number of hugepages:
```
parameter_defaults:
  ContrailDpdkHugepages1GB: 10
```

The snippets below only shows the relevant section of the NIC configuration for each mode.

#### Standard
```
- type: contrail_vrouter_dpdk
  name: vhost0
  use_dhcp: false
  driver: uio_pci_generic
  cpu_list: 0x01
  members:
    -
      type: interface
      name: nic2
      use_dhcp: false
  addresses:
  - ip_netmask:
      get_param: TenantIpSubnet
```

#### VLAN
```
- type: contrail_vrouter_dpdk
  name: vhost0
  use_dhcp: false
  driver: uio_pci_generic
  cpu_list: 0x01
  vlan_id:
    get_param: TenantNetworkVlanID
  members:
    -
      type: interface
      name: nic2
      use_dhcp: false
  addresses:
  - ip_netmask:
      get_param: TenantIpSubnet
```

#### Bond
```
- type: contrail_vrouter_dpdk
  name: vhost0
  use_dhcp: false
  driver: uio_pci_generic
  cpu_list: 0x01
  bond_mode: 4
  bond_policy: layer2+3
  members:
    -
      type: interface
      name: nic2
      use_dhcp: false
    -
      type: interface
      name: nic3
      use_dhcp: false
  addresses:
  - ip_netmask:
      get_param: TenantIpSubnet
```

#### Bond + VLAN
```
- type: contrail_vrouter_dpdk
  name: vhost0
  use_dhcp: false
  driver: uio_pci_generic
  cpu_list: 0x01
  vlan_id:
    get_param: TenantNetworkVlanID
  bond_mode: 4
  bond_policy: layer2+3
  members:
    -
      type: interface
      name: nic2
      use_dhcp: false
    -
      type: interface
      name: nic3
      use_dhcp: false
  addresses:
  - ip_netmask:
      get_param: TenantIpSubnet
```

Advanced vRouter SRIOV Mode Configurations

vRouter SRIOV can be used in the following combinations:

#### SRIOV + Kernel mode
- Standard
- VLAN
- Bond
- Bond + VLAN

#### SRIOV + DPDK mode
- Standard
- VLAN
- Bond
- Bond + VLAN

Network environment configuration

vi ~/tripleo-heat-templates/environments/contrail/contrail-services.yaml
Enable the number of hugepages

SRIOV + Kernel mode
```
parameter_defaults:
  ContrailSriovHugepages1GB: 10
```

SRIOV + DPDK mode
```
parameter_defaults:
  ContrailSriovMode: dpdk
  ContrailDpdkHugepages1GB: 10
  ContrailSriovHugepages1GB: 10
```

SRIOV PF/VF settings
```
NovaPCIPassthrough:
- devname: "ens2f1"
  physical_network: "sriov1"
ContrailSriovNumVFs: ["ens2f1:7"]
```

The SRIOV NICs are not configured in the NIC templates. However, vRouter NICs must still be configured.
So please refer adove NIC template configuration.
