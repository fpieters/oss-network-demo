# Created by Topology-Converter v4.6.9
#    Template Revision: v4.6.9
#    https://github.com/cumulusnetworks/topology_converter
#    using topology data from: ../oss-network-demo/switches/demo.dot
#    built with the following args: ./topology_converter.py ../oss-network-demo/switches/demo.dot -c
#
#    NOTE: in order to use this Vagrantfile you will need:
#       -Vagrant(v2.0.2+) installed: http://www.vagrantup.com/downloads
#       -the "helper_scripts" directory that comes packaged with topology-converter.py
#       -Virtualbox installed: https://www.virtualbox.org/wiki/Downloads



Vagrant.require_version ">= 2.0.2"

# Fix for Older versions of Vagrant to Grab Images from the Correct Location
unless Vagrant::DEFAULT_SERVER_URL.frozen?
  Vagrant::DEFAULT_SERVER_URL.replace('https://vagrantcloud.com')
end

$script = <<-SCRIPT
if grep -q -i 'cumulus' /etc/lsb-release &> /dev/null; then
    echo "### RUNNING CUMULUS EXTRA CONFIG ###"
    source /etc/lsb-release
    if [ -z /etc/app-release ]; then
        echo "  INFO: Detected NetQ TS Server"
        source /etc/app-release
        echo "  INFO: Running NetQ TS Appliance Version $APPLIANCE_VERSION"
    else
        if [[ $DISTRIB_RELEASE =~ ^2.* ]]; then
            echo "  INFO: Detected a 2.5.x Based Release"

            echo "  adding fake cl-acltool..."
            echo -e "#!/bin/bash\nexit 0" > /usr/bin/cl-acltool
            chmod 755 /usr/bin/cl-acltool

            echo "  adding fake cl-license..."
            echo -e "#!/bin/bash\nexit 0" > /usr/bin/cl-license
            chmod 755 /usr/bin/cl-license

            echo "  Disabling default remap on Cumulus VX..."
            mv -v /etc/init.d/rename_eth_swp /etc/init.d/rename_eth_swp.backup

            echo "### Rebooting to Apply Remap..."
        elif [[ $DISTRIB_RELEASE =~ ^3.* ]]; then
            echo "  INFO: Detected a 3.x Based Release ($DISTRIB_RELEASE)"
            echo "### Disabling default remap on Cumulus VX..."
            mv -v /etc/hw_init.d/S10rename_eth_swp.sh /etc/S10rename_eth_swp.sh.backup &> /dev/null
            echo "  INFO: Detected Cumulus Linux v$DISTRIB_RELEASE Release"
            if [[ $DISTRIB_RELEASE =~ ^3.[1-9].* ]]; then
                echo "### Fixing ONIE DHCP to avoid Vagrant Interface ###"
                echo "     Note: Installing from ONIE will undo these changes."
                mkdir /tmp/foo
                mount LABEL=ONIE-BOOT /tmp/foo
                sed -i 's/eth0/eth1/g' /tmp/foo/grub/grub.cfg
                sed -i 's/eth0/eth1/g' /tmp/foo/onie/grub/grub-extra.cfg
                umount /tmp/foo
            fi
            if [[ $DISTRIB_RELEASE =~ ^3.2.* ]]; then
                if [[ $(grep "vagrant" /etc/netd.conf | wc -l ) == 0 ]]; then
                    echo "### Giving Vagrant User Ability to Run NCLU Commands ###"
                    sed -i 's/users_with_edit = root, cumulus/users_with_edit = root, cumulus, vagrant/g' /etc/netd.conf
                    sed -i 's/users_with_show = root, cumulus/users_with_show = root, cumulus, vagrant/g' /etc/netd.conf
                fi
            elif [[ $DISTRIB_RELEASE =~ ^3.[3-9].* ]]; then
                echo "### Giving Vagrant User Ability to Run NCLU Commands ###"
                adduser vagrant netedit
                adduser vagrant netshow
            fi
            echo "### Disabling ZTP service..."
            systemctl stop ztp.service
            ztp -d 2>&1
            echo "### Resetting ZTP to work next boot..."
            ztp -R 2>&1
            ztp -i 2>&1
        fi
    fi
fi
echo "### DONE ###"
echo "### Rebooting Device to Apply Remap..."
nohup bash -c 'sleep 10; shutdown now -r "Rebooting to Remap Interfaces"' &
SCRIPT

Vagrant.configure("2") do |config|

  simid = 1551972815

  config.vm.provider "virtualbox" do |v|
    v.gui=false

  end




  ##### DEFINE VM for oob-mgmt-server #####
  config.vm.define "oob-mgmt-server" do |device|
    
    device.vm.hostname = "oob-mgmt-server" 
    
    device.vm.box = "yk0/ubuntu-xenial"
    device.vm.provider "virtualbox" do |v|
      v.name = "#{simid}_oob-mgmt-server"
      v.customize ["modifyvm", :id, '--audiocontroller', 'AC97', '--audio', 'Null']
      v.memory = 1024
    end
    #   see note here: https://github.com/pradels/vagrant-libvirt#synced-folders
    device.vm.synced_folder ".", "/vagrant", disabled: true



    # NETWORK INTERFACES
      # link for eth1 --> oob-mgmt-switch:swp1
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net20", auto_config: false , :mac => "443839000026"
      

    device.vm.provider "virtualbox" do |vbox|
      vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-all', "--nictype2", "virtio"]
      vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]
    end

    # Fixes "stdin: is not a tty" and "mesg: ttyname failed : Inappropriate ioctl for device"  messages --> https://github.com/mitchellh/vagrant/issues/1673
    device.vm.provision :shell , inline: "(sudo grep -q 'mesg n' /root/.profile 2>/dev/null && sudo sed -i '/mesg n/d' /root/.profile  2>/dev/null) || true;", privileged: false

    # Shorten Boot Process - Applies to Ubuntu Only - remove \"Wait for Network\"
    device.vm.provision :shell , inline: "sed -i 's/sleep [0-9]*/sleep 1/' /etc/init/failsafe.conf 2>/dev/null || true"

    #Copy over DHCP files and MGMT Network Files
    device.vm.provision "file", source: "./helper_scripts/auto_mgmt_network/dhcpd.conf", destination: "~/dhcpd.conf"
    device.vm.provision "file", source: "./helper_scripts/auto_mgmt_network/dhcpd.hosts", destination: "~/dhcpd.hosts"
    device.vm.provision "file", source: "./helper_scripts/auto_mgmt_network/hosts", destination: "~/hosts"
    device.vm.provision "file", source: "./helper_scripts/auto_mgmt_network/ansible_hostfile", destination: "~/ansible_hostfile"
    device.vm.provision "file", source: "./helper_scripts/auto_mgmt_network/ztp_oob.sh", destination: "~/ztp_oob.sh"

    # Run the Config specified in the Node Attributes
    device.vm.provision :shell , privileged: false, :inline => 'echo "$(whoami)" > /tmp/normal_user'
    device.vm.provision :shell , path: "./helper_scripts/auto_mgmt_network/OOB_Server_Config_auto_mgmt.sh"


    # Install Rules for the interface re-map
    device.vm.provision :shell , :inline => <<-delete_udev_directory
if [ -d "/etc/udev/rules.d/70-persistent-net.rules" ]; then
    rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
fi
rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
delete_udev_directory

device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:26 --> eth1"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:26", NAME="eth1", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     
      device.vm.provision :shell , :inline => <<-vagrant_interface_rule
echo "  INFO: Adding UDEV Rule: Vagrant interface = vagrant"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{ifindex}=="2", NAME="vagrant", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
echo "#### UDEV Rules (/etc/udev/rules.d/70-persistent-net.rules) ####"
cat /etc/udev/rules.d/70-persistent-net.rules
vagrant_interface_rule

# Run Any Platform Specific Code and Apply the interface Re-map
    #   (may or may not perform a reboot depending on platform)
    device.vm.provision :shell , :inline => $script

end

  ##### DEFINE VM for oob-mgmt-switch #####
  config.vm.define "oob-mgmt-switch" do |device|
    
    device.vm.hostname = "oob-mgmt-switch" 
    
    device.vm.box = "CumulusCommunity/cumulus-vx"
    device.vm.provider "virtualbox" do |v|
      v.name = "#{simid}_oob-mgmt-switch"
      v.customize ["modifyvm", :id, '--audiocontroller', 'AC97', '--audio', 'Null']
      v.memory = 512
    end
    #   see note here: https://github.com/pradels/vagrant-libvirt#synced-folders
    device.vm.synced_folder ".", "/vagrant", disabled: true



    # NETWORK INTERFACES
      # link for swp1 --> oob-mgmt-server:eth1
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net20", auto_config: false , :mac => "443839000025"
      
      # link for swp2 --> workstation2:eth0
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net21", auto_config: false , :mac => "443839000027"
      
      # link for swp3 --> workstation1:eth0
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net22", auto_config: false , :mac => "443839000029"
      
      # link for swp4 --> spine1:eth0
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net23", auto_config: false , :mac => "44383900002b"
      
      # link for swp5 --> leaf4:eth0
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net24", auto_config: false , :mac => "44383900002d"
      
      # link for swp6 --> spine3:eth0
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net25", auto_config: false , :mac => "44383900002f"
      
      # link for swp7 --> spine2:eth0
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net26", auto_config: false , :mac => "443839000031"
      
      # link for swp8 --> leaf1:eth0
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net27", auto_config: false , :mac => "443839000033"
      
      # link for swp9 --> leaf3:eth0
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net28", auto_config: false , :mac => "443839000035"
      
      # link for swp10 --> leaf2:eth0
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net29", auto_config: false , :mac => "443839000037"
      
      # link for swp11 --> firewall1:eth0
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net30", auto_config: false , :mac => "443839000039"
      
      # link for swp12 --> firewall3:eth0
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net31", auto_config: false , :mac => "44383900003b"
      
      # link for swp13 --> firewall2:eth0
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net32", auto_config: false , :mac => "44383900003d"
      

    device.vm.provider "virtualbox" do |vbox|
      vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-all', "--nictype2", "virtio"]
      vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-all', "--nictype3", "virtio"]
      vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-all', "--nictype4", "virtio"]
      vbox.customize ['modifyvm', :id, '--nicpromisc5', 'allow-all', "--nictype5", "virtio"]
      vbox.customize ['modifyvm', :id, '--nicpromisc6', 'allow-all', "--nictype6", "virtio"]
      vbox.customize ['modifyvm', :id, '--nicpromisc7', 'allow-all', "--nictype7", "virtio"]
      vbox.customize ['modifyvm', :id, '--nicpromisc8', 'allow-all', "--nictype8", "virtio"]
      vbox.customize ['modifyvm', :id, '--nicpromisc9', 'allow-all', "--nictype9", "virtio"]
      vbox.customize ['modifyvm', :id, '--nicpromisc10', 'allow-all', "--nictype10", "virtio"]
      vbox.customize ['modifyvm', :id, '--nicpromisc11', 'allow-all', "--nictype11", "virtio"]
      vbox.customize ['modifyvm', :id, '--nicpromisc12', 'allow-all', "--nictype12", "virtio"]
      vbox.customize ['modifyvm', :id, '--nicpromisc13', 'allow-all', "--nictype13", "virtio"]
      vbox.customize ['modifyvm', :id, '--nicpromisc14', 'allow-all', "--nictype14", "virtio"]
      vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]
    end

    # Fixes "stdin: is not a tty" and "mesg: ttyname failed : Inappropriate ioctl for device"  messages --> https://github.com/mitchellh/vagrant/issues/1673
    device.vm.provision :shell , inline: "(sudo grep -q 'mesg n' /root/.profile 2>/dev/null && sudo sed -i '/mesg n/d' /root/.profile  2>/dev/null) || true;", privileged: false

    #Copy over Topology.dot File
    device.vm.provision "file", source: "../oss-network-demo/switches/demo.dot", destination: "~/topology.dot"
    device.vm.provision :shell, privileged: false, inline: "sudo mv ~/topology.dot /etc/ptm.d/topology.dot"

# Transfer Bridge File
      device.vm.provision "file", source: "./helper_scripts/auto_mgmt_network/bridge-untagged", destination: "~/bridge-untagged"

    # Run the Config specified in the Node Attributes
    device.vm.provision :shell , privileged: false, :inline => 'echo "$(whoami)" > /tmp/normal_user'
    device.vm.provision :shell , path: "./helper_scripts/oob_switch_config.sh"


    # Install Rules for the interface re-map
    device.vm.provision :shell , :inline => <<-delete_udev_directory
if [ -d "/etc/udev/rules.d/70-persistent-net.rules" ]; then
    rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
fi
rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
delete_udev_directory

device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:25 --> swp1"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:25", NAME="swp1", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:27 --> swp2"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:27", NAME="swp2", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:29 --> swp3"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:29", NAME="swp3", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:2b --> swp4"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:2b", NAME="swp4", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:2d --> swp5"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:2d", NAME="swp5", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:2f --> swp6"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:2f", NAME="swp6", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:31 --> swp7"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:31", NAME="swp7", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:33 --> swp8"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:33", NAME="swp8", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:35 --> swp9"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:35", NAME="swp9", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:37 --> swp10"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:37", NAME="swp10", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:39 --> swp11"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:39", NAME="swp11", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:3b --> swp12"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:3b", NAME="swp12", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:3d --> swp13"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:3d", NAME="swp13", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     
      device.vm.provision :shell , :inline => <<-vagrant_interface_rule
echo "  INFO: Adding UDEV Rule: Vagrant interface = vagrant"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{ifindex}=="2", NAME="vagrant", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
echo "#### UDEV Rules (/etc/udev/rules.d/70-persistent-net.rules) ####"
cat /etc/udev/rules.d/70-persistent-net.rules
vagrant_interface_rule

# Run Any Platform Specific Code and Apply the interface Re-map
    #   (may or may not perform a reboot depending on platform)
    device.vm.provision :shell , :inline => $script

end

  ##### DEFINE VM for spine1 #####
  config.vm.define "spine1" do |device|
    
    device.vm.hostname = "spine1" 
    
    device.vm.box = "CumulusCommunity/cumulus-vx"
    device.vm.provider "virtualbox" do |v|
      v.name = "#{simid}_spine1"
      v.customize ["modifyvm", :id, '--audiocontroller', 'AC97', '--audio', 'Null']
      v.memory = 768
    end
    #   see note here: https://github.com/pradels/vagrant-libvirt#synced-folders
    device.vm.synced_folder ".", "/vagrant", disabled: true



    # NETWORK INTERFACES
      # link for eth0 --> oob-mgmt-switch:swp4
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net23", auto_config: false , :mac => "44383900002c"
      
      # link for swp1 --> spine2:swp47
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net16", auto_config: false , :mac => "44383900001f"
      
      # link for swp2 --> spine3:swp47
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net5", auto_config: false , :mac => "443839000009"
      
      # link for swp47 --> firewall1:swp1
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net17", auto_config: false , :mac => "443839000021"
      

    device.vm.provider "virtualbox" do |vbox|
      vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-all', "--nictype2", "virtio"]
      vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-all', "--nictype3", "virtio"]
      vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-all', "--nictype4", "virtio"]
      vbox.customize ['modifyvm', :id, '--nicpromisc5', 'allow-all', "--nictype5", "virtio"]
      vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]
    end

    # Fixes "stdin: is not a tty" and "mesg: ttyname failed : Inappropriate ioctl for device"  messages --> https://github.com/mitchellh/vagrant/issues/1673
    device.vm.provision :shell , inline: "(sudo grep -q 'mesg n' /root/.profile 2>/dev/null && sudo sed -i '/mesg n/d' /root/.profile  2>/dev/null) || true;", privileged: false

    #Copy over Topology.dot File
    device.vm.provision "file", source: "../oss-network-demo/switches/demo.dot", destination: "~/topology.dot"
    device.vm.provision :shell, privileged: false, inline: "sudo mv ~/topology.dot /etc/ptm.d/topology.dot"


    # Run the Config specified in the Node Attributes
    device.vm.provision :shell , privileged: false, :inline => 'echo "$(whoami)" > /tmp/normal_user'
    device.vm.provision :shell , path: "./helper_scripts/extra_switch_config.sh"


    # Install Rules for the interface re-map
    device.vm.provision :shell , :inline => <<-delete_udev_directory
if [ -d "/etc/udev/rules.d/70-persistent-net.rules" ]; then
    rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
fi
rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
delete_udev_directory

device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:2c --> eth0"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:2c", NAME="eth0", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:1f --> swp1"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:1f", NAME="swp1", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:09 --> swp2"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:09", NAME="swp2", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:21 --> swp47"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:21", NAME="swp47", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     
      device.vm.provision :shell , :inline => <<-vagrant_interface_rule
echo "  INFO: Adding UDEV Rule: Vagrant interface = vagrant"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{ifindex}=="2", NAME="vagrant", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
echo "#### UDEV Rules (/etc/udev/rules.d/70-persistent-net.rules) ####"
cat /etc/udev/rules.d/70-persistent-net.rules
vagrant_interface_rule

# Run Any Platform Specific Code and Apply the interface Re-map
    #   (may or may not perform a reboot depending on platform)
    device.vm.provision :shell , :inline => $script

end

  ##### DEFINE VM for spine3 #####
  config.vm.define "spine3" do |device|
    
    device.vm.hostname = "spine3" 
    
    device.vm.box = "CumulusCommunity/cumulus-vx"
    device.vm.provider "virtualbox" do |v|
      v.name = "#{simid}_spine3"
      v.customize ["modifyvm", :id, '--audiocontroller', 'AC97', '--audio', 'Null']
      v.memory = 768
    end
    #   see note here: https://github.com/pradels/vagrant-libvirt#synced-folders
    device.vm.synced_folder ".", "/vagrant", disabled: true



    # NETWORK INTERFACES
      # link for eth0 --> oob-mgmt-switch:swp6
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net25", auto_config: false , :mac => "443839000030"
      
      # link for swp1 --> leaf2:swp52
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net1", auto_config: false , :mac => "443839000001"
      
      # link for swp45 --> firewall3:swp0
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net7", auto_config: false , :mac => "44383900000d"
      
      # link for swp46 --> firewall3:swp1
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net14", auto_config: false , :mac => "44383900001b"
      
      # link for swp47 --> spine1:swp2
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net5", auto_config: false , :mac => "44383900000a"
      
      # link for swp49 --> spine2:swp49
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net9", auto_config: false , :mac => "443839000012"
      
      # link for swp50 --> spine2:swp50
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net18", auto_config: false , :mac => "443839000024"
      

    device.vm.provider "virtualbox" do |vbox|
      vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-all', "--nictype2", "virtio"]
      vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-all', "--nictype3", "virtio"]
      vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-all', "--nictype4", "virtio"]
      vbox.customize ['modifyvm', :id, '--nicpromisc5', 'allow-all', "--nictype5", "virtio"]
      vbox.customize ['modifyvm', :id, '--nicpromisc6', 'allow-all', "--nictype6", "virtio"]
      vbox.customize ['modifyvm', :id, '--nicpromisc7', 'allow-all', "--nictype7", "virtio"]
      vbox.customize ['modifyvm', :id, '--nicpromisc8', 'allow-all', "--nictype8", "virtio"]
      vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]
    end

    # Fixes "stdin: is not a tty" and "mesg: ttyname failed : Inappropriate ioctl for device"  messages --> https://github.com/mitchellh/vagrant/issues/1673
    device.vm.provision :shell , inline: "(sudo grep -q 'mesg n' /root/.profile 2>/dev/null && sudo sed -i '/mesg n/d' /root/.profile  2>/dev/null) || true;", privileged: false

    #Copy over Topology.dot File
    device.vm.provision "file", source: "../oss-network-demo/switches/demo.dot", destination: "~/topology.dot"
    device.vm.provision :shell, privileged: false, inline: "sudo mv ~/topology.dot /etc/ptm.d/topology.dot"


    # Run the Config specified in the Node Attributes
    device.vm.provision :shell , privileged: false, :inline => 'echo "$(whoami)" > /tmp/normal_user'
    device.vm.provision :shell , path: "./helper_scripts/extra_switch_config.sh"


    # Install Rules for the interface re-map
    device.vm.provision :shell , :inline => <<-delete_udev_directory
if [ -d "/etc/udev/rules.d/70-persistent-net.rules" ]; then
    rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
fi
rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
delete_udev_directory

device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:30 --> eth0"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:30", NAME="eth0", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:01 --> swp1"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:01", NAME="swp1", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:0d --> swp45"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:0d", NAME="swp45", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:1b --> swp46"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:1b", NAME="swp46", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:0a --> swp47"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:0a", NAME="swp47", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:12 --> swp49"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:12", NAME="swp49", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:24 --> swp50"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:24", NAME="swp50", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     
      device.vm.provision :shell , :inline => <<-vagrant_interface_rule
echo "  INFO: Adding UDEV Rule: Vagrant interface = vagrant"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{ifindex}=="2", NAME="vagrant", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
echo "#### UDEV Rules (/etc/udev/rules.d/70-persistent-net.rules) ####"
cat /etc/udev/rules.d/70-persistent-net.rules
vagrant_interface_rule

# Run Any Platform Specific Code and Apply the interface Re-map
    #   (may or may not perform a reboot depending on platform)
    device.vm.provision :shell , :inline => $script

end

  ##### DEFINE VM for spine2 #####
  config.vm.define "spine2" do |device|
    
    device.vm.hostname = "spine2" 
    
    device.vm.box = "CumulusCommunity/cumulus-vx"
    device.vm.provider "virtualbox" do |v|
      v.name = "#{simid}_spine2"
      v.customize ["modifyvm", :id, '--audiocontroller', 'AC97', '--audio', 'Null']
      v.memory = 768
    end
    #   see note here: https://github.com/pradels/vagrant-libvirt#synced-folders
    device.vm.synced_folder ".", "/vagrant", disabled: true



    # NETWORK INTERFACES
      # link for eth0 --> oob-mgmt-switch:swp7
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net26", auto_config: false , :mac => "443839000032"
      
      # link for swp1 --> leaf1:swp52
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net2", auto_config: false , :mac => "443839000003"
      
      # link for swp45 --> firewall2:swp0
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net12", auto_config: false , :mac => "443839000017"
      
      # link for swp46 --> firewall2:swp1
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net10", auto_config: false , :mac => "443839000013"
      
      # link for swp47 --> spine1:swp1
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net16", auto_config: false , :mac => "443839000020"
      
      # link for swp49 --> spine3:swp49
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net9", auto_config: false , :mac => "443839000011"
      
      # link for swp50 --> spine3:swp50
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net18", auto_config: false , :mac => "443839000023"
      

    device.vm.provider "virtualbox" do |vbox|
      vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-all', "--nictype2", "virtio"]
      vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-all', "--nictype3", "virtio"]
      vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-all', "--nictype4", "virtio"]
      vbox.customize ['modifyvm', :id, '--nicpromisc5', 'allow-all', "--nictype5", "virtio"]
      vbox.customize ['modifyvm', :id, '--nicpromisc6', 'allow-all', "--nictype6", "virtio"]
      vbox.customize ['modifyvm', :id, '--nicpromisc7', 'allow-all', "--nictype7", "virtio"]
      vbox.customize ['modifyvm', :id, '--nicpromisc8', 'allow-all', "--nictype8", "virtio"]
      vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]
    end

    # Fixes "stdin: is not a tty" and "mesg: ttyname failed : Inappropriate ioctl for device"  messages --> https://github.com/mitchellh/vagrant/issues/1673
    device.vm.provision :shell , inline: "(sudo grep -q 'mesg n' /root/.profile 2>/dev/null && sudo sed -i '/mesg n/d' /root/.profile  2>/dev/null) || true;", privileged: false

    #Copy over Topology.dot File
    device.vm.provision "file", source: "../oss-network-demo/switches/demo.dot", destination: "~/topology.dot"
    device.vm.provision :shell, privileged: false, inline: "sudo mv ~/topology.dot /etc/ptm.d/topology.dot"


    # Run the Config specified in the Node Attributes
    device.vm.provision :shell , privileged: false, :inline => 'echo "$(whoami)" > /tmp/normal_user'
    device.vm.provision :shell , path: "./helper_scripts/extra_switch_config.sh"


    # Install Rules for the interface re-map
    device.vm.provision :shell , :inline => <<-delete_udev_directory
if [ -d "/etc/udev/rules.d/70-persistent-net.rules" ]; then
    rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
fi
rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
delete_udev_directory

device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:32 --> eth0"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:32", NAME="eth0", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:03 --> swp1"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:03", NAME="swp1", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:17 --> swp45"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:17", NAME="swp45", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:13 --> swp46"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:13", NAME="swp46", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:20 --> swp47"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:20", NAME="swp47", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:11 --> swp49"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:11", NAME="swp49", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:23 --> swp50"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:23", NAME="swp50", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     
      device.vm.provision :shell , :inline => <<-vagrant_interface_rule
echo "  INFO: Adding UDEV Rule: Vagrant interface = vagrant"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{ifindex}=="2", NAME="vagrant", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
echo "#### UDEV Rules (/etc/udev/rules.d/70-persistent-net.rules) ####"
cat /etc/udev/rules.d/70-persistent-net.rules
vagrant_interface_rule

# Run Any Platform Specific Code and Apply the interface Re-map
    #   (may or may not perform a reboot depending on platform)
    device.vm.provision :shell , :inline => $script

end

  ##### DEFINE VM for leaf4 #####
  config.vm.define "leaf4" do |device|
    
    device.vm.hostname = "leaf4" 
    
    device.vm.box = "CumulusCommunity/cumulus-vx"
    device.vm.provider "virtualbox" do |v|
      v.name = "#{simid}_leaf4"
      v.customize ["modifyvm", :id, '--audiocontroller', 'AC97', '--audio', 'Null']
      v.memory = 768
    end
    #   see note here: https://github.com/pradels/vagrant-libvirt#synced-folders
    device.vm.synced_folder ".", "/vagrant", disabled: true



    # NETWORK INTERFACES
      # link for eth0 --> oob-mgmt-switch:swp5
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net24", auto_config: false , :mac => "44383900002e"
      
      # link for swp50 --> leaf3:swp50
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net13", auto_config: false , :mac => "44383900001a"
      
      # link for swp52 --> leaf1:swp49
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net3", auto_config: false , :mac => "443839000006"
      

    device.vm.provider "virtualbox" do |vbox|
      vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-all', "--nictype2", "virtio"]
      vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-all', "--nictype3", "virtio"]
      vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-all', "--nictype4", "virtio"]
      vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]
    end

    # Fixes "stdin: is not a tty" and "mesg: ttyname failed : Inappropriate ioctl for device"  messages --> https://github.com/mitchellh/vagrant/issues/1673
    device.vm.provision :shell , inline: "(sudo grep -q 'mesg n' /root/.profile 2>/dev/null && sudo sed -i '/mesg n/d' /root/.profile  2>/dev/null) || true;", privileged: false

    #Copy over Topology.dot File
    device.vm.provision "file", source: "../oss-network-demo/switches/demo.dot", destination: "~/topology.dot"
    device.vm.provision :shell, privileged: false, inline: "sudo mv ~/topology.dot /etc/ptm.d/topology.dot"


    # Run the Config specified in the Node Attributes
    device.vm.provision :shell , privileged: false, :inline => 'echo "$(whoami)" > /tmp/normal_user'
    device.vm.provision :shell , path: "./helper_scripts/extra_switch_config.sh"


    # Install Rules for the interface re-map
    device.vm.provision :shell , :inline => <<-delete_udev_directory
if [ -d "/etc/udev/rules.d/70-persistent-net.rules" ]; then
    rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
fi
rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
delete_udev_directory

device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:2e --> eth0"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:2e", NAME="eth0", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:1a --> swp50"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:1a", NAME="swp50", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:06 --> swp52"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:06", NAME="swp52", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     
      device.vm.provision :shell , :inline => <<-vagrant_interface_rule
echo "  INFO: Adding UDEV Rule: Vagrant interface = vagrant"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{ifindex}=="2", NAME="vagrant", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
echo "#### UDEV Rules (/etc/udev/rules.d/70-persistent-net.rules) ####"
cat /etc/udev/rules.d/70-persistent-net.rules
vagrant_interface_rule

# Run Any Platform Specific Code and Apply the interface Re-map
    #   (may or may not perform a reboot depending on platform)
    device.vm.provision :shell , :inline => $script

end

  ##### DEFINE VM for leaf1 #####
  config.vm.define "leaf1" do |device|
    
    device.vm.hostname = "leaf1" 
    
    device.vm.box = "CumulusCommunity/cumulus-vx"
    device.vm.provider "virtualbox" do |v|
      v.name = "#{simid}_leaf1"
      v.customize ["modifyvm", :id, '--audiocontroller', 'AC97', '--audio', 'Null']
      v.memory = 768
    end
    #   see note here: https://github.com/pradels/vagrant-libvirt#synced-folders
    device.vm.synced_folder ".", "/vagrant", disabled: true



    # NETWORK INTERFACES
      # link for eth0 --> oob-mgmt-switch:swp8
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net27", auto_config: false , :mac => "443839000034"
      
      # link for swp1 --> workstation1:swp1
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net8", auto_config: false , :mac => "443839000010"
      
      # link for swp49 --> leaf4:swp52
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net3", auto_config: false , :mac => "443839000005"
      
      # link for swp50 --> leaf2:swp50
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net4", auto_config: false , :mac => "443839000007"
      
      # link for swp52 --> spine2:swp1
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net2", auto_config: false , :mac => "443839000004"
      

    device.vm.provider "virtualbox" do |vbox|
      vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-all', "--nictype2", "virtio"]
      vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-all', "--nictype3", "virtio"]
      vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-all', "--nictype4", "virtio"]
      vbox.customize ['modifyvm', :id, '--nicpromisc5', 'allow-all', "--nictype5", "virtio"]
      vbox.customize ['modifyvm', :id, '--nicpromisc6', 'allow-all', "--nictype6", "virtio"]
      vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]
    end

    # Fixes "stdin: is not a tty" and "mesg: ttyname failed : Inappropriate ioctl for device"  messages --> https://github.com/mitchellh/vagrant/issues/1673
    device.vm.provision :shell , inline: "(sudo grep -q 'mesg n' /root/.profile 2>/dev/null && sudo sed -i '/mesg n/d' /root/.profile  2>/dev/null) || true;", privileged: false

    #Copy over Topology.dot File
    device.vm.provision "file", source: "../oss-network-demo/switches/demo.dot", destination: "~/topology.dot"
    device.vm.provision :shell, privileged: false, inline: "sudo mv ~/topology.dot /etc/ptm.d/topology.dot"


    # Run the Config specified in the Node Attributes
    device.vm.provision :shell , privileged: false, :inline => 'echo "$(whoami)" > /tmp/normal_user'
    device.vm.provision :shell , path: "./helper_scripts/extra_switch_config.sh"


    # Install Rules for the interface re-map
    device.vm.provision :shell , :inline => <<-delete_udev_directory
if [ -d "/etc/udev/rules.d/70-persistent-net.rules" ]; then
    rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
fi
rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
delete_udev_directory

device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:34 --> eth0"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:34", NAME="eth0", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:10 --> swp1"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:10", NAME="swp1", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:05 --> swp49"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:05", NAME="swp49", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:07 --> swp50"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:07", NAME="swp50", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:04 --> swp52"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:04", NAME="swp52", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     
      device.vm.provision :shell , :inline => <<-vagrant_interface_rule
echo "  INFO: Adding UDEV Rule: Vagrant interface = vagrant"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{ifindex}=="2", NAME="vagrant", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
echo "#### UDEV Rules (/etc/udev/rules.d/70-persistent-net.rules) ####"
cat /etc/udev/rules.d/70-persistent-net.rules
vagrant_interface_rule

# Run Any Platform Specific Code and Apply the interface Re-map
    #   (may or may not perform a reboot depending on platform)
    device.vm.provision :shell , :inline => $script

end

  ##### DEFINE VM for leaf3 #####
  config.vm.define "leaf3" do |device|
    
    device.vm.hostname = "leaf3" 
    
    device.vm.box = "CumulusCommunity/cumulus-vx"
    device.vm.provider "virtualbox" do |v|
      v.name = "#{simid}_leaf3"
      v.customize ["modifyvm", :id, '--audiocontroller', 'AC97', '--audio', 'Null']
      v.memory = 768
    end
    #   see note here: https://github.com/pradels/vagrant-libvirt#synced-folders
    device.vm.synced_folder ".", "/vagrant", disabled: true



    # NETWORK INTERFACES
      # link for eth0 --> oob-mgmt-switch:swp9
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net28", auto_config: false , :mac => "443839000036"
      
      # link for swp1 --> workstation2:swp1
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net11", auto_config: false , :mac => "443839000016"
      
      # link for swp50 --> leaf4:swp50
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net13", auto_config: false , :mac => "443839000019"
      
      # link for swp52 --> leaf2:swp49
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net6", auto_config: false , :mac => "44383900000c"
      

    device.vm.provider "virtualbox" do |vbox|
      vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-all', "--nictype2", "virtio"]
      vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-all', "--nictype3", "virtio"]
      vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-all', "--nictype4", "virtio"]
      vbox.customize ['modifyvm', :id, '--nicpromisc5', 'allow-all', "--nictype5", "virtio"]
      vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]
    end

    # Fixes "stdin: is not a tty" and "mesg: ttyname failed : Inappropriate ioctl for device"  messages --> https://github.com/mitchellh/vagrant/issues/1673
    device.vm.provision :shell , inline: "(sudo grep -q 'mesg n' /root/.profile 2>/dev/null && sudo sed -i '/mesg n/d' /root/.profile  2>/dev/null) || true;", privileged: false

    #Copy over Topology.dot File
    device.vm.provision "file", source: "../oss-network-demo/switches/demo.dot", destination: "~/topology.dot"
    device.vm.provision :shell, privileged: false, inline: "sudo mv ~/topology.dot /etc/ptm.d/topology.dot"


    # Run the Config specified in the Node Attributes
    device.vm.provision :shell , privileged: false, :inline => 'echo "$(whoami)" > /tmp/normal_user'
    device.vm.provision :shell , path: "./helper_scripts/extra_switch_config.sh"


    # Install Rules for the interface re-map
    device.vm.provision :shell , :inline => <<-delete_udev_directory
if [ -d "/etc/udev/rules.d/70-persistent-net.rules" ]; then
    rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
fi
rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
delete_udev_directory

device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:36 --> eth0"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:36", NAME="eth0", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:16 --> swp1"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:16", NAME="swp1", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:19 --> swp50"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:19", NAME="swp50", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:0c --> swp52"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:0c", NAME="swp52", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     
      device.vm.provision :shell , :inline => <<-vagrant_interface_rule
echo "  INFO: Adding UDEV Rule: Vagrant interface = vagrant"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{ifindex}=="2", NAME="vagrant", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
echo "#### UDEV Rules (/etc/udev/rules.d/70-persistent-net.rules) ####"
cat /etc/udev/rules.d/70-persistent-net.rules
vagrant_interface_rule

# Run Any Platform Specific Code and Apply the interface Re-map
    #   (may or may not perform a reboot depending on platform)
    device.vm.provision :shell , :inline => $script

end

  ##### DEFINE VM for leaf2 #####
  config.vm.define "leaf2" do |device|
    
    device.vm.hostname = "leaf2" 
    
    device.vm.box = "CumulusCommunity/cumulus-vx"
    device.vm.provider "virtualbox" do |v|
      v.name = "#{simid}_leaf2"
      v.customize ["modifyvm", :id, '--audiocontroller', 'AC97', '--audio', 'Null']
      v.memory = 768
    end
    #   see note here: https://github.com/pradels/vagrant-libvirt#synced-folders
    device.vm.synced_folder ".", "/vagrant", disabled: true



    # NETWORK INTERFACES
      # link for eth0 --> oob-mgmt-switch:swp10
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net29", auto_config: false , :mac => "443839000038"
      
      # link for swp49 --> leaf3:swp52
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net6", auto_config: false , :mac => "44383900000b"
      
      # link for swp50 --> leaf1:swp50
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net4", auto_config: false , :mac => "443839000008"
      
      # link for swp52 --> spine3:swp1
      device.vm.network "private_network", virtualbox__intnet: "#{simid}_net1", auto_config: false , :mac => "443839000002"
      

    device.vm.provider "virtualbox" do |vbox|
      vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-all', "--nictype2", "virtio"]
      vbox.customize ['modifyvm', :id, '--nicpromisc3', 'allow-all', "--nictype3", "virtio"]
      vbox.customize ['modifyvm', :id, '--nicpromisc4', 'allow-all', "--nictype4", "virtio"]
      vbox.customize ['modifyvm', :id, '--nicpromisc5', 'allow-all', "--nictype5", "virtio"]
      vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]
    end

    # Fixes "stdin: is not a tty" and "mesg: ttyname failed : Inappropriate ioctl for device"  messages --> https://github.com/mitchellh/vagrant/issues/1673
    device.vm.provision :shell , inline: "(sudo grep -q 'mesg n' /root/.profile 2>/dev/null && sudo sed -i '/mesg n/d' /root/.profile  2>/dev/null) || true;", privileged: false

    #Copy over Topology.dot File
    device.vm.provision "file", source: "../oss-network-demo/switches/demo.dot", destination: "~/topology.dot"
    device.vm.provision :shell, privileged: false, inline: "sudo mv ~/topology.dot /etc/ptm.d/topology.dot"


    # Run the Config specified in the Node Attributes
    device.vm.provision :shell , privileged: false, :inline => 'echo "$(whoami)" > /tmp/normal_user'
    device.vm.provision :shell , path: "./helper_scripts/extra_switch_config.sh"


    # Install Rules for the interface re-map
    device.vm.provision :shell , :inline => <<-delete_udev_directory
if [ -d "/etc/udev/rules.d/70-persistent-net.rules" ]; then
    rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
fi
rm -rfv /etc/udev/rules.d/70-persistent-net.rules &> /dev/null
delete_udev_directory

device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:38 --> eth0"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:38", NAME="eth0", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:0b --> swp49"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:0b", NAME="swp49", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:08 --> swp50"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:08", NAME="swp50", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     device.vm.provision :shell , :inline => <<-udev_rule
echo "  INFO: Adding UDEV Rule: 44:38:39:00:00:02 --> swp52"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="44:38:39:00:00:02", NAME="swp52", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
udev_rule
     
      device.vm.provision :shell , :inline => <<-vagrant_interface_rule
echo "  INFO: Adding UDEV Rule: Vagrant interface = vagrant"
echo 'ACTION=="add", SUBSYSTEM=="net", ATTR{ifindex}=="2", NAME="vagrant", SUBSYSTEMS=="pci"' >> /etc/udev/rules.d/70-persistent-net.rules
echo "#### UDEV Rules (/etc/udev/rules.d/70-persistent-net.rules) ####"
cat /etc/udev/rules.d/70-persistent-net.rules
vagrant_interface_rule

# Run Any Platform Specific Code and Apply the interface Re-map
    #   (may or may not perform a reboot depending on platform)
    device.vm.provision :shell , :inline => $script

end



end