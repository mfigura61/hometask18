# -*- mode: ruby -*-
# vi: set ft=ruby :

disk = './secondDisk.vdi'
Vagrant.configure(2) do |config|
  config.vm.box = "centos/7"


  config.vm.define "server" do |server|
    server.vm.network "private_network", ip: "192.168.10.10"
    server.vm.provider "virtualbox" do |v|
      v.memory = 1024
      v.cpus = 2
    end
    server.vm.hostname = "server"
  end

  config.vm.define "backup" do |backup|
    backup.vm.network "private_network", ip: "192.168.10.20"
    backup.vm.provider "virtualbox" do |v|
      v.memory = 1024
      v.cpus = 2      
  

   unless File.exist?(disk)
        v.customize ['createhd', '--filename', disk, '--variant', 'Fixed', '--size', 10 * 1024]
        end
        v.customize ['storageattach', :id,  '--storagectl', 'IDE', '--port', 1, '--device', 0, '--type', 'hdd', '--medium', disk]
        end

    backup.vm.hostname = "backup"
    config.vm.provision "shell", inline: <<-SHELL
            mkdir -p ~root/.ssh
            cp ~vagrant/.ssh/auth* ~root/.ssh
            
    SHELL
  
      
  end

end
