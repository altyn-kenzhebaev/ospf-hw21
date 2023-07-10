# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
:router1 => {
        :box_name => "ubuntu/jammy64",
        :nets => {
                  :net2 => {
                    :ip => '10.0.10.1', 
                    :adapter => 2, 
                    :netmask => "255.255.255.252", 
                    :virtualbox__intnet => "r1-r2"
                  },
                  :net3 => {
                    :ip => '10.0.12.1', 
                    :adapter => 3, 
                    :netmask => "255.255.255.252", 
                    :virtualbox__intnet => "r1-r3"
                  },
                  :net4 => {
                    :ip => '192.168.10.1', 
                    :adapter => 4, 
                    :netmask => "255.255.255.0", 
                    :virtualbox__intnet => "net1"
                  },
                 }
  },

  :router2 => {
        :box_name => "ubuntu/jammy64",
        :nets => {
                  :net2 => {
                    :ip => '10.0.10.2', 
                    :adapter => 2, 
                    :netmask => "255.255.255.252", 
                    :virtualbox__intnet => "r1-r2"
                  },
                  :net3 => {
                    :ip => '10.0.11.2', 
                    :adapter => 3, 
                    :netmask => "255.255.255.252", 
                    :virtualbox__intnet => "r2-r3"
                  },
                  :net4 => {
                    :ip => '192.168.20.1', 
                    :adapter => 4, 
                    :netmask => "255.255.255.0", 
                    :virtualbox__intnet => "net2"
                  },
                 }
  },

  :router3 => {
    :box_name => "ubuntu/jammy64",
    :nets => {
              :net2 => {
                :ip => '10.0.11.1', 
                :adapter => 2, 
                :netmask => "255.255.255.252", 
                :virtualbox__intnet => "r2-r3"
              },
              :net3 => {
                :ip => '10.0.12.2', 
                :adapter => 3, 
                :netmask => "255.255.255.252", 
                :virtualbox__intnet => "r1-r3"
              },
              :net4 => {
                :ip => '192.168.30.1', 
                :adapter => 4, 
                :netmask => "255.255.255.0", 
                :virtualbox__intnet => "net3"
              },
             }
  },
}

Vagrant.configure("2") do |config|

  MACHINES.each do |boxname, boxconfig|

    config.vm.define boxname do |box|

        box.vm.box = boxconfig[:box_name]
        box.vm.host_name = boxname.to_s

        boxconfig[:nets].each do |netname, netconf|
          box.vm.network "private_network", 
          ip: netconf[:ip], 
          adapter: netconf[:adapter], 
          netmask: netconf[:netmask], 
          virtualbox__intnet: netconf[:virtualbox__intnet]
        end

        if boxname.to_s == "router3"
          box.vm.provision "ansible" do |ansible|
            ansible.playbook = 'ansible/ospf.yml'
            ansible.inventory_path = "ansible/hosts"
            ansible.compatibility_mode = "2.0"
          end
        end

      end

  end
  
end
