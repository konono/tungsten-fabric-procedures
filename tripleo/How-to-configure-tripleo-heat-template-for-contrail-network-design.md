# 今からでもできる！TripleO Heat Template入門 Network 設計編 
## Network configuration plan

### **編集対象ファイル**
- environments/network-isolation.yaml
- environments/ips-from-pool-all.yaml *Option

### network-isolation.yaml
ここでは、それぞれのRoleに対してどのネットワークセグメントを割り振るかを操作できる
例えば、Control NodeではStorage,Storage_mgmt,InternalAPI,ExternalAPI,Tenantのセグメントを割り振っているが、Contrail ControllerにはInternalAPIと、Tenantしか割り振っていない。
このように、使うセグメントと使わないセグメントを設定することで、必要最低限の設定を行うことができる。

NWセグメントの設計はOpenStackへの理解が求められるが、まずは書きスライドを参考にして設計すると良い。

![NW設計参考1](/uploads/NW_diagram01.png "NW設計参考1")

![NW設計参考2](/uploads/NW_diagram01.png "NW設計参考2")

```
# Enable the creation of Neutron networks for isolated Overcloud
# traffic and configure each role to assign ports (related
# to that role) on these networks.
# primary role is: Controller
resource_registry:
  # networks as defined in network_data.yaml
  OS::TripleO::Network::Storage: ../network/storage.yaml
  OS::TripleO::Network::StorageMgmt: ../network/storage_mgmt.yaml
  OS::TripleO::Network::InternalApi: ../network/internal_api.yaml
  OS::TripleO::Network::Tenant: ../network/tenant.yaml
  OS::TripleO::Network::External: ../network/external.yaml
  OS::TripleO::Network::Management: ../network/management.yaml

  # Port assignments for the VIPs
  OS::TripleO::Network::Ports::StorageVipPort: ../network/ports/storage.yaml
  OS::TripleO::Network::Ports::StorageMgmtVipPort: ../network/ports/storage_mgmt.yaml
  OS::TripleO::Network::Ports::InternalApiVipPort: ../network/ports/internal_api.yaml
  OS::TripleO::Network::Ports::ExternalVipPort: ../network/ports/external.yaml
  OS::TripleO::Network::Ports::RedisVipPort: ../network/ports/vip.yaml

  # Port assignments by role, edit role definition to assign networks to roles.
  # Port assignments for the Controller
  OS::TripleO::Controller::Ports::StoragePort: ../network/ports/storage.yaml
  OS::TripleO::Controller::Ports::StorageMgmtPort: ../network/ports/storage_mgmt.yaml
  OS::TripleO::Controller::Ports::InternalApiPort: ../network/ports/internal_api.yaml
  OS::TripleO::Controller::Ports::ExternalPort: ../network/ports/external.yaml
  OS::TripleO::Controller::Ports::TenantPort: ../network/ports/tenant.yaml

  # Port assignments for the Compute
  OS::TripleO::Compute::Ports::StoragePort: ../network/ports/storage.yaml
  OS::TripleO::Compute::Ports::InternalApiPort: ../network/ports/internal_api.yaml
  OS::TripleO::Compute::Ports::TenantPort: ../network/ports/tenant.yaml

  # Port assignments for the ContrailController
  OS::TripleO::ContrailController::Ports::InternalApiPort: ../network/ports/internal_api.yaml
  OS::TripleO::ContrailController::Ports::TenantPort: ../network/ports/tenant.yaml

  # Port assignments for the ContrailControlOnly
  OS::TripleO::ContrailControlOnly::Ports::InternalApiPort: ../network/ports/internal_api.yaml
  OS::TripleO::ContrailControlOnly::Ports::TenantPort: ../network/ports/tenant.yaml

  # Port assignments for the ContrailDpdk
  OS::TripleO::ContrailDpdk::Ports::StoragePort: ../network/ports/storage.yaml
  OS::TripleO::ContrailDpdk::Ports::InternalApiPort: ../network/ports/internal_api.yaml
  OS::TripleO::ContrailDpdk::Ports::TenantPort: ../network/ports/tenant.yaml

  # Port assignments for the ContrailSriov
  OS::TripleO::ContrailSriov::Ports::StoragePort: ../network/ports/storage.yaml
  OS::TripleO::ContrailSriov::Ports::InternalApiPort: ../network/ports/internal_api.yaml
  OS::TripleO::ContrailSriov::Ports::TenantPort: ../network/ports/tenant.yaml

  # Port assignments for the ContrailTsn
  OS::TripleO::ContrailTsn::Ports::StoragePort: ../network/ports/storage.yaml
  OS::TripleO::ContrailTsn::Ports::InternalApiPort: ../network/ports/internal_api.yaml
  OS::TripleO::ContrailTsn::Ports::TenantPort: ../network/ports/tenant.yaml

```

### ips-from-pool-all.yaml
このファイルはオプションで利用することができる。
デフォルトでは、UndercloudのNeutron DHCPからIPアドレスがアサインされるため、IPの固定はできないが、このコンフィギュレーションを利用することで可能になる。
network-isolationでNWをアサインされているが、IPアドレスを振りたくない場合は、../network/ports/noop.yamlと指定することで、IPアドレッシングを回避することができる。

```
cat environments/ips-from-pool-all.yaml
# Environment file demonstrating how to pre-assign IPs to all node types
resource_registry:
  OS::TripleO::ContrailController::Ports::ExternalPort: ../network/ports/noop.yaml
  OS::TripleO::ContrailController::Ports::InternalApiPort: ../network/ports/internal_api_from_pool.yaml
  OS::TripleO::ContrailController::Ports::TenantPort: ../network/ports/tenant_from_pool.yaml
  OS::TripleO::ContrailController::Ports::StoragePort: ../network/ports/noop.yaml
  OS::TripleO::ContrailController::Ports::StorageMgmtPort: ../network/ports/noop.yaml

  OS::TripleO::ContrailAnalytics::Ports::ExternalPort: ../network/ports/external_from_pool.yaml
  OS::TripleO::ContrailAnalytics::Ports::InternalApiPort: ../network/ports/internal_api_from_pool.yaml
  OS::TripleO::ContrailAnalytics::Ports::TenantPort: ../network/ports/tenant_from_pool.yaml
  OS::TripleO::ContrailAnalytics::Ports::StoragePort: ../network/ports/storage_from_pool.yaml
  OS::TripleO::ContrailAnalytics::Ports::StorageMgmtPort: ../network/ports/storage_mgmt_from_pool.yaml

  OS::TripleO::ContrailAnalyticsDatabase::Ports::ExternalPort: ../network/ports/external_from_pool.yaml
  OS::TripleO::ContrailAnalyticsDatabase::Ports::InternalApiPort: ../network/ports/internal_api_from_pool.yaml
  OS::TripleO::ContrailAnalyticsDatabase::Ports::TenantPort: ../network/ports/tenant_from_pool.yaml
  OS::TripleO::ContrailAnalyticsDatabase::Ports::StoragePort: ../network/ports/storage_from_pool.yaml
  OS::TripleO::ContrailAnalyticsDatabase::Ports::StorageMgmtPort: ../network/ports/storage_mgmt_from_pool.yaml

  OS::TripleO::Controller::Ports::ExternalPort: ../network/ports/noop.yaml
  OS::TripleO::Controller::Ports::InternalApiPort: ../network/ports/internal_api_from_pool.yaml
  OS::TripleO::Controller::Ports::StoragePort: ../network/ports/noop.yaml
  OS::TripleO::Controller::Ports::StorageMgmtPort: ../network/ports/noop.yaml
  OS::TripleO::Controller::Ports::TenantPort: ../network/ports/tenant_from_pool.yaml
  # Management network is optional and disabled by default
  #OS::TripleO::Controller::Ports::ManagementPort: ../network/ports/management_from_pool.yaml

  OS::TripleO::Compute::Ports::ExternalPort: ../network/ports/noop.yaml
  OS::TripleO::Compute::Ports::InternalApiPort: ../network/ports/internal_api_from_pool.yaml
  OS::TripleO::Compute::Ports::StoragePort: ../network/ports/noop.yaml
  OS::TripleO::Compute::Ports::StorageMgmtPort: ../network/ports/noop.yaml
  OS::TripleO::Compute::Ports::TenantPort: ../network/ports/tenant_from_pool.yaml
  #OS::TripleO::Compute::Ports::ManagementPort: ../network/ports/management_from_pool.yaml

  OS::TripleO::CephStorage::Ports::ExternalPort: ../network/ports/noop.yaml
  OS::TripleO::CephStorage::Ports::InternalApiPort: ../network/ports/noop.yaml
  OS::TripleO::CephStorage::Ports::StoragePort: ../network/ports/storage_from_pool.yaml
  OS::TripleO::CephStorage::Ports::StorageMgmtPort: ../network/ports/storage_mgmt_from_pool.yaml
  OS::TripleO::CephStorage::Ports::TenantPort: ../network/ports/noop.yaml
  #OS::TripleO::CephStorage::Ports::ManagementPort: ../network/ports/management_from_pool.yaml

  OS::TripleO::ObjectStorage::Ports::ExternalPort: ../network/ports/noop.yaml
  OS::TripleO::ObjectStorage::Ports::InternalApiPort: ../network/ports/internal_api_from_pool.yaml
  OS::TripleO::ObjectStorage::Ports::StoragePort: ../network/ports/storage_from_pool.yaml
  OS::TripleO::ObjectStorage::Ports::StorageMgmtPort: ../network/ports/storage_mgmt_from_pool.yaml
  OS::TripleO::ObjectStorage::Ports::TenantPort: ../network/ports/noop.yaml
  #OS::TripleO::ObjectStorage::Ports::ManagementPort: ../network/ports/management_from_pool.yaml

  OS::TripleO::BlockStorage::Ports::ExternalPort: ../network/ports/noop.yaml
  OS::TripleO::BlockStorage::Ports::InternalApiPort: ../network/ports/internal_api_from_pool.yaml
  OS::TripleO::BlockStorage::Ports::StoragePort: ../network/ports/storage_from_pool.yaml
  OS::TripleO::BlockStorage::Ports::StorageMgmtPort: ../network/ports/storage_mgmt_from_pool.yaml
  OS::TripleO::BlockStorage::Ports::TenantPort: ../network/ports/noop.yaml
  #OS::TripleO::BlockStorage::Ports::ManagementPort: ../network/ports/management_from_pool.yaml

### Roleが複数台アサインされている場合は、IPアドレスを複数書くことで表現が可能。
###
###  ContrailControllerIPs:
###    # Each controller will get an IP from the lists below, first controller, first IP
###    ctlplane:
###    - 172.16.1.55
###    - 172.16.1.56
###    - 172.16.1.57
###    internal_api:
###    - 172.27.116.55
###    - 172.27.116.56
###    - 172.27.116.57
###    tenant:
###    - 172.16.4.55
###    - 172.16.4.56
###    - 172.16.4.57

parameter_defaults:
  ContrailControllerIPs:
    # Each controller will get an IP from the lists below, first controller, first IP
    ctlplane:
    - 172.16.1.55
    internal_api:
    - 172.27.116.55
    tenant:
    - 172.16.4.55
  ContrailAnalyticsIPs:
    # Each controller will get an IP from the lists below, first controller, first IP
    external:
    - 172.27.115.50
    internal_api:
    - 192.168.1.12
    storage:
    - 10.3.0.12
    storage_mgmt:
    - 10.4.0.12
    tenant:
    - 10.84.50.222
    #management:
    #- 172.16.4.251
  ContrailAnalyticsDatabaseIPs:
    # Each controller will get an IP from the lists below, first controller, first IP
    external:
    - 172.27.115.51
    internal_api:
    - 192.168.1.13
    storage:
    - 10.3.0.13
    storage_mgmt:
    - 10.4.0.13
    tenant:
    - 10.84.50.223
    #management:
  ControllerIPs:
    # Each controller will get an IP from the lists below, first controller, first IP
    ctlplane:
    - 172.16.1.56
    internal_api:
    - 172.27.116.56
    tenant:
    - 172.16.4.56
  ComputeIPs:
    # Each compute will get an IP from the lists below, first compute, first IP
    ctlplane:
    - 172.16.1.57
    internal_api:
    - 172.27.116.57
    tenant:
    - 172.16.4.57
  CephStorageIPs:
    # Each ceph node will get an IP from the lists below, first node, first IP
    storage:
    - 172.16.1.253
    storage_mgmt:
    - 172.16.3.253
    #management:
    #- 172.16.4.253
  SwiftStorageIPs:
    # Each swift node will get an IP from the lists below, first node, first IP
    internal_api:
    - 172.16.2.254
    storage:
    - 172.16.1.254
    storage_mgmt:
    - 172.16.3.254
    #management:
    #- 172.16.4.254
  BlockStorageIPs:
    # Each cinder node will get an IP from the lists below, first node, first IP
    internal_api:
    - 172.16.2.250
    storage:
    - 172.16.1.250
    storage_mgmt:
    - 172.16.3.250
    #management:
    #- 172.16.4.250

### VIPのアサインは下記のように実施する、VIPの設定が必要になルものはここで操作する。
### NIC Templateでアサインされていないものについてはコメントアウトしておくと良い。
  # Predictable VIPs
  ControlFixedIPs: [{'ip_address':'172.16.1.82'}] # undercloudのdhcp start, end の pool外から指定
  InternalApiVirtualFixedIPs: [{'ip_address':'172.27.116.82'}] # pools外から指定
  # PublicVirtualFixedIPs: [{'ip_address':'172.16.23.4'}] # pools外から指定
  # StorageVirtualFixedIPs: [{'ip_address':'172.18.0.9'}] # pools外から指定
  # StorageMgmtVirtualFixedIPs: [{'ip_address':'172.19.0.9'}] # pools外から指定
  RedisVirtualFixedIPs: [{'ip_address':'172.27.116.83'}] # pools外から指定
  #ManagementVirtualFixedIPs: [{'ip_address':'172.16.21.4'}]
  #OverProvVirtualFixedIPs: [{'ip_address':'10.243.15.4'}]
```
