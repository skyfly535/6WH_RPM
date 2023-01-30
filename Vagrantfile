# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
  config.vm.box = "centos/7"
  config.vm.box_version = "2004.01"

  config.vm.provider "virtualbox" do |v|
    v.memory = 256
    v.cpus = 1
  end

  config.vm.define "repos" do |repos|
    repos.vm.network "private_network", ip: "192.168.56.10", virtualbox__intnet: "net1"
    repos.vm.hostname = "repoServer"
    repos.vm.provision "shell", path: "repos_script.sh"
  end

  config.vm.define "repoc" do |repoc|
    repoc.vm.network "private_network", ip: "192.168.56.11", virtualbox__intnet: "net1"
    repoc.vm.hostname = "repoClient"
    repoc.vm.provision "shell", path: "repoc_script.sh"
  end

end
