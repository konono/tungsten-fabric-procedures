# contrailデプロイ手順

# Contrail 5.0.2 + OpenStack TripleO Simple Deploy with kernel vRouter

## 0. Rqeuirement

### Over Cloud:

OS: Red Hat Enterprise Linux Server release 7.5 (Maipo)

Kernel: 3.10.0-693.21.1.el7.x86_64

Network Interface: 3NIC

eth0: Ironic PXE boot

eth1: InternalAPI/ExternalAPI (DefaultGW)

eth2: Tenant 

ContrailImageTag: 5.0.2-0.360-rhel-queens

### Under Cloud:

OS: Red Hat Enterprise Linux Server release 7.6 (Maipo)

Kernel: 3.10.0-862.14.4.el7.x86_64

Network Interface: 3NIC

eth1: InternalAPI/ExternalAPI (DefaultGW)

eth2: Ironic PXE boot

eth5: Tenant 

internalAPI: 172.27.116.0/23

ironic: 172.16.1.0/24

tenant: 172.16.4.0/24

## 1. Under cloud deploy
Target: **UnderCloud**

### 1.1 subscribe rhn
```
sudo subscription-manager register
sudo subscription-manager subscribe
sudo subscription-manager repos --disable=*
sudo subscription-manager repos --enable=rhel-7-server-rpms --enable=rhel-7-server-extras-rpms --enable=rhel-7-server-rh-common-rpms --enable=rhel-ha-for-rhel-7-server-rpms --enable=rhel-7-server-openstack-13-rpms --enable=rhel-7-server-openstack-13-devtools-rpms
```

### 1.2 Package install and Kernel update
```
yum -y update
yum install -y libguestfs \
 libguestfs-tools \
 qemu-kvm \
 libvirt \
 python-virtinst \
 python-tripleoclient \
 tmux \
 vim \
 git

sudo reboot
```

### 1.3 Prepare install undercloud
```
useradd stack
mkdir /home/stack/images
chown stack. /home/stack/images
echo "stack ALL=(root) NOPASSWD:ALL" | tee -a /etc/sudoers.d/stack
chmod 440 /etc/sudoers.d/stack
passwd stack

sudo su - stack

cat << EOF > /home/stack/undercloud.conf
[DEFAULT]
local_ip = 172.16.1.148/24
network_gateway = 172.16.1.148
undercloud_public_vip = 172.16.1.1
undercloud_admin_vip = 172.16.1.2
local_interface = eth2
network_cidr = 172.16.1.0/24
masquerade_network = 172.16.1.0/24
dhcp_start = 172.16.1.20
dhcp_end = 172.16.1.50
inspection_interface = br-ctlplane
inspection_iprange = 172.16.1.201,172.16.1.225
inspection_extras = true
inspection_runbench = false
undercloud_debug = true
enable_tempest = true
ipxe_deploy = true
store_events = false
[auth]
EOF
```

### 1.4 Install undercloud
```
sudo -u stack openstack undercloud install
```

## 2. Deploy overcloud 

### 2.1 Configure nameserver for overcloud nodes
Target: **UnderCloud**
```
cat << EOF > setup_dns.sh
#!/bin/bash

source ~/stackrc
Undercloud_nameserver=8.8.8.8
openstack subnet set `openstack subnet show ctlplane-subnet -c id -f value` --dns-nameserver ${Undercloud_nameserver}
EOF
bash -x setup_dns.sh
```

### 2.2 Setup overcloud images
#### 2.2.1 Install overcloud images and Unpack overcloud image

```
sudo yum install -y rhosp-director-images rhosp-director-images-ipa

cat <<EOF > unpack_tripleo_base_image.sh
#!/bin/bash
for i in /usr/share/rhosp-director-images/overcloud-full-latest-13.0.tar /usr/share/rhosp-director-images/ironic-python-agent-latest-13.0.tar ; do sudo tar -xvf $i -C /home/stack/images/; done
EOF
bash -x unpack_tripleo_base_image.sh

cat <<EOF > upload_tripleo_base_image.sh
#!/bin/bash

source ~/stackrc
openstack overcloud image upload --update-existing --image-path /home/stack/images/
bash -x upload_tripleo_base_image.sh
```

#### 2.2.2 Mod overcloud image
```

sudo -i 
export LIBGUESTFS_BACKEND=direct
cd /home/stack/images/
virt-customize --root-password password:lab -a overcloud-full.qcow2
```

### 2.3 Prepare overcloud vm nodes
Target: **KVM Node**

#### 2.3.1 create bridge and configure network

Internet bridge can reach internet.

Plz "internet bridge" setup adjust for your environment.

```
cat <<EOF > create_br.sh
#!/bin/bash

sudo brctl addbr ironic
sudo brctl addbr tenant

sudo ip addr add 172.16.1.10/24 dev ironic
sudo ip addr add 172.16.4.10/24 dev tenant

sudo ifconfig ironic up
sudo ifconfig tenant up

ping 172.16.1.1 -c 5
ping 172.16.4.1 -c 5
EOF
bash -x create_br.sh
```

#### 2.3.2 create vm and virtualBMC setup
```
sudo pip install virtualbmc

cat <<EOF > cvm_osp13.sh
#!/bin/bash

if (( $EUID != 0 )); then
    echo "Please run as root"
    exit
fi

num=0
ipmi_user=root
ipmi_password=lab
libvirt_path=/var/lib/libvirt/images
port_group=Overcloud
prov_switch=ironic
ROLES=compute:1,contrail-controller:1,control:1

/bin/rm ironic_list
IFS=',' read -ra role_list <<< "${ROLES}"for role in ${role_list[@]}; do
  echo $role
  role_name=`echo $role|cut -d ":" -f 1`
  role_count=`echo $role|cut -d ":" -f 2`
  for count in `seq 1 ${role_count}`; do
    echo $role_name $count
    sudo qemu-img create -f qcow2 ${libvirt_path}/${role_name}_${count}.qcow2 99G
    sudo virsh define /dev/stdin << EOF
    $(virt-install --name ${role_name}_${count} \
    --disk ${libvirt_path}/${role_name}_${count}.qcow2 \
    --vcpus=4 \
    --ram=16348 \
    --network bridge=ironic,model=virtio,portgroup=${port_group} \
    --network bridge=internet,model=virtio \
    --network bridge=tenant,model=virtio \
    --virt-type kvm \
    --cpu host \
    --import \
    --os-variant rhel7 \
    --serial pty \
    --console pty,target_type=virtio \
    --graphics vnc,listen=0.0.0.0,password=l\
    --noautoconsole \
    --print-xml)
EOF

    sudo vbmc add ${role_name}_${count} --port 1623${num} --username ${ipmi_user} --password ${ipmi_password} --libvirt-uri=qemu:///session
    sudo vbmc start ${role_name}_${count}
    prov_mac=`sudo virsh domiflist ${role_name}_${count}|grep ${prov_switch}|awk '{print $5}'`
    vm_name=${role_name}-${count}-`hostname -s`
    kvm_ip=`ip route get 1 |grep src |awk '{print $7}'`
    echo ${prov_mac} ${vm_name} ${kvm_ip} ${role_name} 1623${num}>> ironic_list
    num=$(expr $num + 1)
  done
done
EOF
bash -x cvm_osp13.sh

scp ./ironic_list stack@[undercloud_ip]:~/
```

### 2.4 Add virtual Server to Ironic
Target: **undercloud**
```
cat <<EOF > def_vms.sh
#!/bin/bash

source ~/stackrc

kvm_ip=172.27.116.102
ipmi_user=root
ipmi_password=lab

while IFS= read -r line; do
  mac=`echo $line|awk '{print $1}'`
  name=`echo $line|awk '{print $2}'`
  kvm_ip=`echo $line|awk '{print $3}'`
  profile=`echo $line|awk '{print $4}'`
  ipmi_port=`echo $line|awk '{print $5}'`
  uuid=`openstack baremetal node create --driver ipmi \
                                        --property cpus=4 \
                                        --property memory_mb=16348 \
                                        --property local_gb=100 \
                                        --property cpu_arch=x86_64 \
                                        --driver-info ipmi_username=${ipmi_user}  \
                                        --driver-info ipmi_address=${kvm_ip} \
                                        --driver-info ipmi_password=${ipmi_password} \
                                        --driver-info ipmi_port=${ipmi_port} \
                                        --name=${name} \
                                        --property capabilities=profile:${profile},boot_option:local \
                                        -c uuid -f value`
  openstack baremetal port create --node ${uuid} ${mac}
done < <(cat ironic_list)

DEPLOY_KERNEL=$(openstack image show bm-deploy-kernel -f value -c id)
DEPLOY_RAMDISK=$(openstack image show bm-deploy-ramdisk -f value -c id)

for i in `openstack baremetal node list -c UUID -f value`; do
  openstack baremetal node set $i --driver-info deploy_kernel=$DEPLOY_KERNEL --driver-info deploy_ramdisk=$DEPLOY_RAMDISK
done

for i in `openstack baremetal node list -c UUID -f value`; do
  openstack baremetal node show $i -c properties -f value
done
EOF

bash -x def_vms.sh
```

### 2.5 Introspection 
```
cat <<EOF > introspection.sh
#!/bin/bash
source ~/stackrc
for node in $(openstack baremetal node list -c UUID -f value) ; do
  openstack baremetal node manage $node
done
openstack overcloud node introspect --all-manageable --provide
EOF
bash -x introspection.sh
```

### 2.6 Create Flavoar
```
cat <<EOF > create_flavor.sh
#!/bin/bash
source ../stackrc
for i in compute-dpdk compute-sriov contrail-controller contrail-analytics contrail-database contrail-analytics-database; do
  openstack flavor create $i --ram 4096 --vcpus 1 --disk 40
  openstack flavor set --property "capabilities:boot_option"="local" \
                       --property "capabilities:profile"="${i}" ${i}
done
EOF
bash -x create_flavor.sh
```

### 2.7 Create heat template
```
cat <<EOF > template_setup.sh
cp -r /usr/share/openstack-tripleo-heat-templates/ tripleo-heat-templates.origin
git clone https://github.com/juniper/contrail-tripleo-heat-templates -b stable/queens
cp -r contrail-tripleo-heat-templates/* tripleo-heat-templates.origin/
EOF
bash -x template_setup.sh

cp -a tripleo-heat-templates.origin/ tripleo-heat-templates
cd tripleo-heat-templates/
git init
git commit -m 'Initial commit'
```

### 2.8 Create OpenStack container file
```
cat <<EOF > upload_container_image.sh
#!/bin/bash

source ../stackrc

tag=13.0-91
undercloud_ip=$(netstat -tnl |grep 8787 |awk '{print $4}'|awk -F: '{print $1}')
version=`echo $tag|awk -F'-' '{print $1}'`
release=`echo $tag|awk -F'-' '{print $2}'`

openstack overcloud container image prepare \
 --push-destination=${undercloud_ip}:8787  \
 --tag-from-label ${version}-${release} \
 --output-images-file ~/local_registry_images.yaml  \
 --namespace=registry.access.redhat.com/rhosp13 \
 --prefix=openstack-  \
 --tag-from-label ${version}-${release}  \
 --output-env-file ~/overcloud_images.yaml

openstack overcloud container image upload --config-file ~/local_registry_images.yaml --verbose
EOF
bash -x upload_container_image.sh

mkdir ~/tripleo-heat-templates/user-configuration
cp -p ~/overcloud_images.yaml ~/tripleo-heat-templates/user-configuration/
```

### 2.9 Configure tripleO-heate-template

#### [TripleO NIC Template](How-to-configure-tripleo-heat-template-for-NIC-Template)
#### [TripleO contrail services](How-to-configure-tripleo-heat-template-for-contrail-services)
#### [TripleO contrail NW Design](How-to-configure-tripleo-heat-template-for-contrail-network-design)
 
### 2.10 Install overcloud
```
mkdir ~/tripleo-heat-templates/sh
mkdir ~/tripleo-heat-templates/sh/log
cd ~/tripleo-heat-templates/sh

cat <<EOF > overcloud-deploy.sh
#! /bin/bash

source ~/stackrc
tht=/home/stack/tripleo-heat-templates.test
now_date=`date +%m%d-%H-%M%S`

time openstack overcloud deploy --templates ${tht}/ \
--roles-file ${tht}/user-configuration/roles_data_contrail_aio.yaml \
-e ${tht}/environments/network-isolation.yaml \
-e ${tht}/environments/contrail/contrail-plugins.yaml \
-e ${tht}/user-configuration/overcloud_images.yaml \
-e ${tht}/user-configuration/parameter.yaml \
-e ${tht}/environments/contrail/contrail-net.yaml \
-e ${tht}/environments/contrail/contrail-services.yaml \
-e ${tht}/environments/ips-from-pool-all.yaml \
--ntp-server ntp.jst.mfeed.ad.jp \
--stack overcloud  2>&1 |tee ./log/overcloud_deploy_${now_date}.log

if [ $? -ne 0 ];then
  echo '###openstack stack list --nested###' > ./log/deployment_TS_${now_date}.log
  openstack stack list --nested >> ./log/deployment_TS_${now_date}.log
  echo '###openstack stack failures list overcloud###' >> ./log/deployment_TS_${now_date}.log
  openstack stack failures list overcloud >> ./log/deployment_TS_${now_date}.log
  echo '###openstack stack show ###' >> ./log/deployment_TS_${now_date}.log
  openstack stack list --nested |grep -v COMP | awk '{print $2}'|grep -v -e ID -e ^$|xargs -I% openstack stack show % >> ./log/deployment_TS_${now_date}.log
fi
EOF

bash -x overcloud-deploy.sh

Enjoy Contrail :)