# Install Vagrant-libvirt

### Ubuntu 12.04/14.04/16.04, Debian:
```
$ wget https://releases.hashicorp.com/vagrant/2.0.3/vagrant_2.0.3_x86_64.deb
$ sudo dpkg -i vagrant_2.0.3_x86_64.deb
$ sudo apt-get install qemu libvirt-bin ebtables ruby-libvirt 
$ sudo apt-get install libxslt-dev libxml2-dev libvirt-dev zlib1g-dev ruby-dev
```

### Install vagrant-libvirt plugin
```
$ vagrant plugin install vagrant-libvirt
$ vagrant plugin install vagrant-mutate
```

### Add Box
```
$ vagrant box add centos/7
```

### Create Quickstart Vagrantfile
And then make a Vagrantfile that looks like the following, filling in your information where necessary. For example:

```
Vagrant.configure("2") do |config|
  config.vm.define :test_vm do |test_vm|
    test_vm.vm.box = "fedora/24-cloud-base"
  end
end
```

### Start VM
In prepared project directory, run following command:

```
$ vagrant up --provider=libvirt
```

### Stop VM
```
$ vagrant halt
```

### Destroy VM
```
$ vagrant destroy
```

### Sample Vagrantfile
```
------------------------------------------------------------------------
# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure(2) do |config|
  # The most common configuration options are documented and commented below.
  # For a complete reference, please see the online documentation at
  # https://docs.vagrantup.com.

  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://vagrantcloud.com/search.
  config.vm.box = "homeski/rhel7.3-osp"

  # Disable automatic box update checking. If you disable this, then
  # boxes will only be checked for updates when the user runs
  # `vagrant box outdated`. This is not recommended.
  # config.vm.box_check_update = false

  config.vm.define :osp1 do |osp1|
    osp1.vm.network :public_network,
      ip: "172.27.114.151",
      netmask: "255.255.252.0",
      dev: "internet",
      mode: "bridge",
      type: "bridge"

    osp1.vm.network :public_network,
      ip: "172.27.114.152",
      netmask: "255.255.252.0",
      dev: "internet",
      mode: "bridge",
      type: "bridge"

    osp1.vm.provider :libvirt do |lv|
      lv.uri = 'qemu+unix:///system'
      #lv.host = "remote.host.com"
      lv.cpus = 4
      lv.memory = 2048
      lv.boot 'network'
      lv.boot 'hd'
      lv.management_network_name = "vagrant-libvirt"
    end
    osp1.vm.provision "shell", run: "always", inline: <<-EOF
      route del default
      route add default gw 172.27.112.1
    EOF
  end


### PXE usecase
  config.vm.define :pxeclient do |pxeclient|
    pxeclient.vm.network :forwarded_port, guest: 80, host: 2000, host_ip: "0.0.0.0"

### Port forward
#netstat -tnl
#Active Internet connections (only servers)
#Proto Recv-Q Send-Q Local Address           Foreign Address         State
#tcp        0      0 127.0.0.1:5900          0.0.0.0:*               LISTEN
#tcp        0      0 0.0.0.0:2000            0.0.0.0:*               LISTEN

#curl localhost:2000
#<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 3.2 Final//EN"><html>
#<title>Directory listing for /</title>
#<body>
#<h2>Directory listing for /</h2>
#<hr>
#<ul>
#<li><a href=".bash_history">.bash_history</a>
#<li><a href=".bash_logout">.bash_logout</a>
#<li><a href=".bash_profile">.bash_profile</a>
#<li><a href=".bashrc">.bashrc</a>
#<li><a href=".ssh/">.ssh/</a>
#</ul>
#<hr>
#</body>
#</html>

    pxeclient.vm.network :private_network,
      ip: '10.0.0.5',
      libvirt__network_name: "am",
      libvirt__dhcp_enabled: false,
      libvirt__host_ip: '10.0.0.1',
      libvirt__forward_mode: 'nat',
      libvirt__forward_device: 'internet'
      ### dump xml
        #  <forward dev='virbr100' mode='nat'>
        #-A POSTROUTING -s 10.0.0.0/24 ! -d 10.0.0.0/24 -o virbr100 -p tcp -j MASQUERADE --to-ports 1024-65535
        #-A POSTROUTING -s 10.0.0.0/24 ! -d 10.0.0.0/24 -o virbr100 -p udp -j MASQUERADE --to-ports 1024-65535
        #-A POSTROUTING -s 10.0.0.0/24 ! -d 10.0.0.0/24 -o virbr100 -j MASQUERADE

      #libvirt__iface_name: 'virbr100'
        #change veth name vnet[x] => virbr100

### dump xml
#<network connections='1' ipv6='yes'>
#  <name>am</name>
#  <uuid>f69220c8-51db-44c4-837b-f8e7853aac9a</uuid>
#  <forward mode='nat'>
#    <nat>
#      <port start='1024' end='65535'/>
#    </nat>
#  </forward>
#  <bridge name='virbr0' stp='on' delay='0'/>
#  <mac address='52:54:00:96:69:8d'/>
#  <ip address='10.0.0.1' netmask='255.255.255.0'>
#  </ip>
#</network>


    pxeclient.vm.network :private_network,
      ip: '10.1.0.5',
      libvirt__network_name: "bm",
      libvirt__dhcp_enabled: false,
      libvirt__host_ip: '10.1.0.1',
      libvirt__forward_mode: 'route'

### dump xml
#<network connections='1' ipv6='yes'>
#  <name>bm</name>
#  <uuid>88cd613b-7b0c-457d-808a-0cced91cdc9a</uuid>
#  <forward mode='route'/>
#  <bridge name='virbr1' stp='on' delay='0'/>
#  <mac address='52:54:00:3d:97:fa'/>
#  <ip address='10.1.0.1' netmask='255.255.255.0'>
#  </ip>
#</network>

    pxeclient.vm.network :private_network,
      ip: '10.2.0.5',
      libvirt__network_name: "cm",
      libvirt__dhcp_enabled: false,
      libvirt__host_ip: '10.2.0.1',
      libvirt__forward_mode: 'none'

#<network connections='1' ipv6='yes'>
#  <name>cm</name>
#  <uuid>42955e12-ef83-4bbf-a23a-641fc3d091a5</uuid>
#  <bridge name='virbr2' stp='on' delay='0'/>
#  <mac address='52:54:00:69:8b:d2'/>
#  <ip address='10.2.0.1' netmask='255.255.255.0'>
#  </ip>
#</network>

    pxeclient.vm.provider :libvirt do |lv|
      lv.uri = 'qemu+unix:///system'
      lv.cpus = 4
      lv.memory = 1000
      boot_network = {'network' => 'am'}
      lv.storage :file, :size => '100G', :type => 'qcow2'
      lv.boot boot_network
      lv.boot 'hd'
      lv.management_network_name = "vagrant-libvirt"
    end
  end

  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine. In the example below,
  # accessing "localhost:8080" will access port 80 on the guest machine.
  # config.vm.network "forwarded_port", guest: 80, host: 8080

  # Create a private network, which allows host-only access to the machine
  # using a specific IP.
  # config.vm.network "private_network", ip: "192.168.33.10"

  # Create a public network, which generally matched to bridged network.
  # Bridged networks make the machine appear as another physical device on
  # your network.

end
------------------------------------------------------------------------
```

# TIPS

### How to change private network bridge name
```
$ vim /home/ubuntu/.vagrant.d/gems/gems/vagrant-libvirt-0.0.43/lib/vagrant-libvirt/action/create_networks.rb
#:277     while lookup_bridge_by_name(bridge_name = "virbr#{count}") <--- You can change this name to whatever you like.
```

### Bug information

[Can't create "public bridged network" together with "private network" in a VM.](https://github.com/vagrant-libvirt/vagrant-libvirt/issues/842)


Happy Vagrant :)
