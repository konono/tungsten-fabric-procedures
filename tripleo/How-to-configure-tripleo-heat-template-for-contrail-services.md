# 今からでもできる！TripleO Heat Template入門 Contrail Service編
## NIC Template Configuration

### **編集対象ファイル**
- environments/contrail/contrail-services.yaml


### contrail-services.yaml
ここでは、Contrailに対してConfigurationを渡したり、どのサービスでどのSegmentを使うかなどの設定が行える。

```
resource_registry:
  OS::TripleO::Services::ContrailConfigDatabase: ../../docker/services/contrail/contrail-config-database.yaml
  OS::TripleO::Services::ContrailAnalyticsDatabase: ../../docker/services/contrail/contrail-analytics-database.yaml
  OS::TripleO::Services::ContrailConfig: ../../docker/services/contrail/contrail-config.yaml
  OS::TripleO::Services::ContrailAnalytics: ../../docker/services/contrail/contrail-analytics.yaml
  OS::TripleO::Services::ContrailControl: ../../docker/services/contrail/contrail-control.yaml
  OS::TripleO::Services::ContrailWebui: ../../docker/services/contrail/contrail-webui.yaml
  OS::TripleO::Services::ContrailVrouter: ../../docker/services/contrail/contrail-vrouter.yaml
  OS::TripleO::Services::ContrailDpdk: ../../docker/services/contrail/contrail-vrouter-dpdk.yaml
  OS::TripleO::Services::NeutronCorePlugin: ../../docker/services/contrail/contrail-neutron-container-plugin.yaml
  OS::TripleO::Services::ContrailHeatPlugin: ../../docker/services/contrail/contrail-heat-container-plugin.yaml
  OS::TripleO::Services::ComputeNeutronCorePlugin: ../../docker/services/contrail/contrail-vrouter.yaml
  OS::TripleO::Compute::PreNetworkConfig: ../../extraconfig/pre_network/contrail/compute_pre_network.yaml
  OS::TripleO::ContrailDpdk::PreNetworkConfig: ../../extraconfig/pre_network/contrail/contrail_dpdk_pre_network.yaml

parameter_defaults:
### ServiceNetMapはどのサービスに対してどのセグメントを割り振るかを設定できる、今回はExternalAPIとInternlAPIを統合したかったので、例として下記のように変更した
#######
#PublicNetwork: external -> PublicNetwork: internal_api
#######
  ServiceNetMap:
    ApacheNetwork: internal_api
    NeutronTenantNetwork: tenant
    CeilometerApiNetwork: internal_api
    ContrailAnalyticsNetwork: internal_api
    ContrailAnalyticsDatabaseNetwork: internal_api
    ContrailConfigNetwork: internal_api
#    ContrailControlNetwork: tenant
    ContrailControlNetwork: internal_api
    ContrailDatabaseNetwork: internal_api
    ContrailWebuiNetwork: internal_api
    ContrailTsnNetwork: internal_api
#    ContrailVrouterNetwork: tenant
    AodhApiNetwork: internal_api
    PankoApiNetwork: internal_api
    BarbicanApiNetwork: internal_api
    GnocchiApiNetwork: internal_api
    MongodbNetwork: internal_api
    CinderApiNetwork: internal_api
    CinderIscsiNetwork: storage
    CongressApiNetwork: internal_api
    GlanceApiNetwork: internal_api
    IronicApiNetwork: over_prov 
    IronicNetwork: over_prov 
    IronicInspectorNetwork: over_prov 
    KeystoneAdminApiNetwork: ctlplane # allows undercloud to config endpoints
    KeystonePublicApiNetwork: internal_api
    ManilaApiNetwork: internal_api
    NeutronApiNetwork: internal_api
    OctaviaApiNetwork: internal_api
    HeatApiNetwork: internal_api
    HeatApiCfnNetwork: internal_api
    HeatApiCloudwatchNetwork: internal_api
    NovaApiNetwork: internal_api
    NovaColdMigrationNetwork: ctlplane
    NovaPlacementNetwork: internal_api
    NovaMetadataNetwork: internal_api
    NovaVncProxyNetwork: internal_api
    # NovaVncProxyNetwork: management
    NovaLibvirtNetwork: internal_api
    Ec2ApiNetwork: internal_api
    Ec2ApiMetadataNetwork: internal_api
    TackerApiNetwork: internal_api
    #SwiftStorageNetwork: storage # Changed from storage_mgmt
    SwiftStorageNetwork: tenant
    #SwiftProxyNetwork: storage
    SwiftProxyNetwork: internal_api
    SaharaApiNetwork: internal_api
    HorizonNetwork: internal_api
    # HorizonNetwork: management
    MemcachedNetwork: internal_api
    RabbitmqNetwork: internal_api
    QdrNetwork: internal_api
    RedisNetwork: internal_api
    MysqlNetwork: internal_api
    CephClusterNetwork: storage # Changed from storage_mgmt
    CephMonNetwork: storage
    CephRgwNetwork: storage
    #PublicNetwork: external
    PublicNetwork: internal_api
    OpendaylightApiNetwork: internal_api
    OvnDbsNetwork: internal_api
    MistralApiNetwork: internal_api
    ZaqarApiNetwork: internal_api
    PacemakerRemoteNetwork: internal_api
    EtcdNetwork: internal_api
    CephStorageHostnameResolveNetwork: storage
    ControllerHostnameResolveNetwork: internal_api
    ComputeHostnameResolveNetwork: internal_api
    ObjectStorageHostnameResolveNetwork: internal_api
    BlockStorageHostnameResolveNetwork: internal_api
### どのRoleでどのFlavorを使うかの指定がなされている
  OvercloudControllerFlavor: control
  OvercloudContrailControllerFlavor: contrail-controller
  OvercloudContrailControlOnlyFlavor: control-only
  OvercloudComputeFlavor: compute
  OvercloudContrailDpdkFlavor: compute-dpdk
  OvercloudContrailSriovFlavor: compute-sriov
  OvercloudContrailAnalyticsFlavor: contrail-analytics
### Roleごとにデプロイする台数を指定、Red Hat的推奨台数は一度のデプロイで50台までらしい
  ControllerCount: 1
  ContrailControllerCount: 1
  ContrailControlOnlyCount: 0
  ContrailAnalyticsCount: 0
  ComputeCount: 1
  ContrailDpdkCount: 0
  ContrailSriovCount: 0

  NeutronMetadataProxySharedSecret: secret
# enable public juniper registry
### Contrailのデプロイで使用するレジストリと、タグの設定
  ContrailRegistry: hub.juniper.net/contrail
  ContrailRegistryUser: xxxxxxxxxxxxxxxxx
  ContrailRegistryPassword: xxxxxxxxxxxxxx
  ContrailImageTag: 5.0.2-0.360-rhel-queens
### ContrailSettingsではcommon_contrail.envの中に記述されるconfigurationをベタでここに書くことができる
  ContrailSettings:
    VROUTER_GATEWAY: 172.16.4.1
    CONFIG_NODEMGR__DEFAULTS__minimum_diskGB: 2
    DATABASE_NODEMGR__DEFAULTS__minimum_diskGB: 2
    CONFIG_DATABASE_NODEMGR__DEFAULTS__minimum_diskGB: 2
```
